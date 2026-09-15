#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
steam_watch.py — Steam 評論 / 討論版定期監控 agent
==================================================

針對《幻世錄 重製版》(本體 4030150 / 試玩版 4778780) 抓取：
  1. 玩家評論      Steam 官方 appreviews API（含分頁 cursor、多語言）
  2. 評分摘要      各語言 total / positive / negative / 評價等級（精準、極省流量）
  3. 討論版主題    社群討論版列表（標題、作者、回覆數、連結）

資料存進 SQLite，每次執行只抓新的，並且比對上次結果產出「異動摘要」：
新評論、新負評、評分變化、討論串新回覆。

用法
----
  python steam_watch.py init                 # 建立資料庫
  python steam_watch.py fetch --full         # 第一次：抓全部歷史評論
  python steam_watch.py fetch                # 之後每天：只抓新的（增量）
  python steam_watch.py report               # 產出 dashboard.html + report.md
  python steam_watch.py run --full           # fetch + report 一次做完
  python steam_watch.py seed reviews.jsonl --appid 4778780   # 匯入既有資料（測試用）

排程（Windows 工作排程器）
  程式: python.exe   引數: C:\\path\\steam_watch.py run   起始位置: C:\\path
排程（macOS / Linux crontab，每天 10:00）
  0 10 * * * cd /path && /usr/bin/python3 steam_watch.py run >> watch.log 2>&1

選項
  --webhook <URL>    把異動摘要 POST 到 Discord / Slack webhook
  --langs a,b,c      要單獨追蹤評分的語言（預設 tchinese,schinese,english,japanese,koreana）
  --no-threads       這次不抓討論版
  --db / --out       資料庫與輸出目錄

只依賴標準函式庫（有 requests 就用，沒有就用 urllib）。
"""

import argparse
import json
import os
import re
import sqlite3
import sys
import time
import urllib.parse
import urllib.request
from datetime import datetime, timedelta, timezone

# ---------------------------------------------------------------- 設定 --------

APPS = {
    4030150: "幻世錄 重製版",          # 正式版：評論與討論版都在這個 hub
}
# 試玩版 4778780 已停止追蹤（2026-09-10 起），不抓、也不納入任何報表
DISCUSSION_APPS = [4030150]          # 討論版集中在本體 hub
DEFAULT_LANGS = ["tchinese", "schinese", "english", "japanese", "koreana"]
LANG_LABEL = {
    "tchinese": "繁中", "schinese": "簡中", "english": "英文",
    "japanese": "日文", "koreana": "韓文", "thai": "泰文",
    "russian": "俄文", "spanish": "西文", "german": "德文",
    "french": "法文", "portuguese": "葡文", "polish": "波蘭文",
}
TW = timezone(timedelta(hours=8))
UA = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) steam-watch/1.0"

# 主題關鍵詞：改這裡就能換成你要追的議題
THEMES = [
    ("AI 素材 / AI 味",   ["ai味", "ai 味", "ai味儿", "ai味兒", "ai立绘", "ai立繪", "ai配音",
                           "ai生成", "ai跑", "ai 生成", "ai-generated", "ai generated",
                           "ai art", "generative", "ai素材", "ai製作", "ai制作", "aiっぽ", "aiちっく", "ai 떡칠"]),
    ("配音 / 語音",        ["配音", "语音", "語音", "声优", "聲優", "音声", "음성", "voice", "voice over", "voiceover"]),
    ("命中率 / MISS",      ["miss", "命中", "闪避", "閃避", "回避"]),
    ("數值 / 平衡",        ["数值", "數值", "平衡", "难度", "難度", "balance", "難易度", "sl", "回溯"]),
    ("UI / 介面",          ["ui", "介面", "界面", "布局", "佈局"]),
    ("BGM / 音樂",         ["bgm", "音乐", "音樂", "编曲", "編曲", "音效", "브금", "music"]),
    ("畫面 / 美術",        ["画质", "畫質", "画面", "畫面", "美工", "美术", "美術", "立绘", "立繪",
                            "建模", "高清", "画风", "畫風", "art", "graphics", "일러"]),
    ("情懷 / 補票",        ["情怀", "情懷", "补票", "補票", "童年", "老登", "回忆", "回憶", "怀念", "懷念",
                            "nostalg", "思い出", "추억"]),
    ("續作 / 二代",        ["二代", "2代", "幻2", "幻世录2", "幻世錄2", "三代", "3代", "超时空", "超時空", "續作", "续作"]),
    ("卡牌 / 打牌",        ["卡牌", "打牌", "昆特", "card"]),
    ("BUG / 當機",         ["bug", "闪退", "閃退", "卡界面", "black screen", "crash", "当机", "當機", "存档", "存檔"]),
    ("多結局 / IF 劇情",   ["多结局", "多結局", "if剧情", "if劇情", "if线", "if線", "结局", "結局",
                            "意难平", "意難平", "分歧", "路线", "路線", "ending", "エンディング", "ルート"]),
    ("本地化 / 字型顯示",  ["字体", "字型", "字幕", "翻译", "翻譯", "本地化", "繁体", "繁體", "简体", "簡體",
                            "日语显示", "日語顯示", "font", "localization", "translation", "英文太大"]),
    ("退款風險",           ["退款", "退費", "退货", "退貨", "refund", "30分钟", "30分鐘", "返金"]),
    ("發售日 / 價格",      ["发售日", "發售日", "价格", "價格", "售价", "售價", "定价", "定價", "多少钱",
                            "多少錢", "折扣", "打折", "price", "继承", "繼承", "上线", "上線", "发售", "發售"]),
    ("購買意願",           ["必买", "必買", "肯定买", "肯定買", "一定會買", "一定会买", "会买", "會買",
                            "买单", "買單", "期待正式版", "製品版買", "购入", "購入", "will buy", "buying"]),
]

# ------------------------------------------------------------ HTTP 小工具 -----

try:
    import requests  # noqa
    _SESSION = requests.Session()
    _SESSION.headers.update({
        "User-Agent": UA,
        "Accept-Language": "zh-TW,zh;q=0.9,en;q=0.8",
        # 討論版偶爾會擋成年內容／年齡牆，先把 cookie 備好
        "Cookie": "birthtime=628578001; lastagecheckage=1-0-1990; mature_content=1; Steam_Language=tchinese",
    })

    def http_get(url, timeout=30):
        r = _SESSION.get(url, timeout=timeout)
        r.raise_for_status()
        return r.text
except Exception:                                        # pragma: no cover
    def http_get(url, timeout=30):
        req = urllib.request.Request(url, headers={
            "User-Agent": UA,
            "Accept-Language": "zh-TW,zh;q=0.9,en;q=0.8",
            "Cookie": "birthtime=628578001; lastagecheckage=1-0-1990; "
                      "mature_content=1; Steam_Language=tchinese",
        })
        with urllib.request.urlopen(req, timeout=timeout) as resp:
            return resp.read().decode("utf-8", "replace")


def get_json(url, tries=3):
    last = None
    for i in range(tries):
        try:
            return json.loads(http_get(url))
        except Exception as e:                            # 網路/JSON 都在這裡重試
            last = e
            time.sleep(2 * (i + 1))
    raise RuntimeError(f"抓取失敗 {url}: {last}")


# ---------------------------------------------------------------- 資料庫 ------

SCHEMA = """
CREATE TABLE IF NOT EXISTS reviews (
  id TEXT PRIMARY KEY, appid INTEGER, lang TEXT, voted_up INTEGER,
  ts_created INTEGER, ts_updated INTEGER, playtime INTEGER,
  votes_up INTEGER, votes_funny INTEGER, comment_count INTEGER,
  steam_purchase INTEGER, refunded INTEGER, early_access INTEGER,
  text TEXT, author TEXT, first_seen INTEGER
);
CREATE INDEX IF NOT EXISTS idx_rev_app_ts ON reviews(appid, ts_created);
CREATE TABLE IF NOT EXISTS summaries (
  appid INTEGER, lang TEXT, checked_at INTEGER, total INTEGER,
  positive INTEGER, negative INTEGER, score INTEGER, score_desc TEXT,
  PRIMARY KEY (appid, lang, checked_at)
);
CREATE TABLE IF NOT EXISTS threads (
  id TEXT PRIMARY KEY, appid INTEGER, title TEXT, author TEXT,
  replies INTEGER, url TEXT, first_seen INTEGER, last_seen INTEGER
);
-- 討論串的「當日動態」：那天新開串、或那天多了幾則回覆，就算那天的內容
CREATE TABLE IF NOT EXISTS thread_activity (
  tid TEXT, day TEXT, kind TEXT, delta INTEGER, replies INTEGER,
  PRIMARY KEY (tid, day)
);
CREATE INDEX IF NOT EXISTS idx_act_day ON thread_activity(day);
CREATE TABLE IF NOT EXISTS runs (
  id INTEGER PRIMARY KEY AUTOINCREMENT, started_at INTEGER,
  new_reviews INTEGER, new_threads INTEGER, note TEXT
);
"""


def open_db(path):
    os.makedirs(os.path.dirname(os.path.abspath(path)) or ".", exist_ok=True)
    con = sqlite3.connect(path)
    con.row_factory = sqlite3.Row
    con.executescript(SCHEMA)
    return con


# ---------------------------------------------------------------- 抓評論 -----

def review_page(appid, cursor="*", num=100, lang="all", day_range=365):
    q = {
        "json": 1, "filter": "recent", "language": lang, "purchase_type": "all",
        "review_type": "all", "num_per_page": num, "day_range": day_range,
        "cursor": cursor,
    }
    url = f"https://store.steampowered.com/appreviews/{appid}?" + urllib.parse.urlencode(q)
    return get_json(url)


def fetch_reviews(con, appid, full=False, max_pages=200, verbose=True):
    """抓評論。full=False 時，連續遇到已知評論就停（增量模式）。"""
    known = {r[0] for r in con.execute("SELECT id FROM reviews WHERE appid=?", (appid,))}
    now = int(time.time())
    cursor, seen_cursors, new_rows, dup_streak = "*", set(), [], 0

    for page in range(max_pages):
        data = review_page(appid, cursor)
        if not data.get("success"):
            break
        revs = data.get("reviews") or []
        if not revs:
            break
        for r in revs:
            rid = str(r["recommendationid"])
            if rid in known:
                dup_streak += 1
                continue
            dup_streak = 0
            a = r.get("author") or {}
            new_rows.append((
                rid, appid, r.get("language"), 1 if r.get("voted_up") else 0,
                r.get("timestamp_created"), r.get("timestamp_updated"),
                a.get("playtime_at_review") or a.get("playtime_forever") or 0,
                r.get("votes_up", 0), r.get("votes_funny", 0), r.get("comment_count", 0),
                1 if r.get("steam_purchase") else 0, 1 if r.get("refunded") else 0,
                1 if r.get("written_during_early_access") else 0,
                (r.get("review") or "").strip(), a.get("steamid"), now,
            ))
        nxt = data.get("cursor")
        if verbose:
            print(f"  [{appid}] 第 {page+1} 頁：{len(revs)} 筆，累積新增 {len(new_rows)}")
        if not nxt or nxt in seen_cursors:
            break
        seen_cursors.add(nxt)
        cursor = nxt
        if not full and dup_streak >= 40:      # 已經追到上次的位置
            break
        if len(revs) < 100:
            break
        time.sleep(1.0)                        # 對 Steam 客氣一點

    con.executemany(
        "INSERT OR IGNORE INTO reviews VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)", new_rows)
    con.commit()
    return new_rows


def fetch_summaries(con, appid, langs):
    """各語言評分摘要：num_per_page=0，非常便宜，數字是 Steam 官方統計。"""
    now, out = int(time.time()), {}
    for lang in ["all"] + list(langs):
        try:
            d = review_page(appid, num=0, lang=lang)
            s = d.get("query_summary") or {}
            out[lang] = s
            con.execute(
                "INSERT OR REPLACE INTO summaries VALUES (?,?,?,?,?,?,?,?)",
                (appid, lang, now, s.get("total_reviews", 0), s.get("total_positive", 0),
                 s.get("total_negative", 0), s.get("review_score", 0),
                 s.get("review_score_desc", "")))
        except Exception as e:
            print(f"  ! 摘要抓取失敗 {appid}/{lang}: {e}")
        time.sleep(0.4)
    con.commit()
    return out


# -------------------------------------------------------------- 抓討論版 -----

def _grab(block, pattern):
    m = re.search(pattern, block, re.S)
    return m.group(1) if m else ""


def parse_topics(html, appid):
    """把討論版列表頁解析成 [(tid, title, author, replies, url)]。

    Steam 改版過好幾次、各欄位順序也不固定，所以這裡不寫死一條長 regex：
    先用 forum_topic 區塊切開，每塊各自撈欄位；真的切不開再退回「掃連結」。
    """
    url_re = re.compile(
        r'href="(https://steamcommunity\.com/app/%d/discussions/\d+/(\d+)/?[^"]*)"' % appid)
    found, order = {}, []

    def add(tid, title, author, replies, url):
        if tid in found:
            return
        found[tid] = (tid, strip_tags(title).strip(), strip_tags(author).strip(),
                      int(re.sub(r"[^\d]", "", replies or "0") or 0), url.split("?")[0])
        order.append(tid)

    for blk in re.split(r'(?=<div[^>]+class="[^"]*forum_topic[ "])', html):
        m = url_re.search(blk)
        if not m:
            continue
        title = (_grab(blk, r'class="forum_topic_name"[^>]*>(.*?)</div>') or
                 _grab(blk, r'forum_topic_name[^>]*>(.*?)<\s*/') or
                 _grab(blk, r'<a[^>]+forum_topic_overlay[^>]*>(.*?)</a>'))
        author = (_grab(blk, r'class="forum_topic_op"[^>]*>(.*?)</div>') or
                  _grab(blk, r'forum_topic_op[^>]*>(.*?)<\s*/'))
        replies = (_grab(blk, r'class="forum_topic_reply_count"[^>]*>\s*([\d,]+)') or
                   _grab(blk, r'forum_topic_reply_count[^>]*>\s*([\d,]+)'))
        add(m.group(2), title, author, replies, m.group(1))

    if not found:                      # 退路：至少把標題跟連結撈出來
        for m in url_re.finditer(html):
            tail = html[m.end():m.end() + 900]
            title = (_grab(tail, r'forum_topic_name[^>]*>(.*?)</div>') or
                     _grab(tail, r'>([^<>]{4,120})<'))
            add(m.group(2), title, "", _grab(tail, r'forum_topic_reply_count[^>]*>\s*([\d,]+)'),
                m.group(1))
    return [found[t] for t in order]


def strip_tags(s):
    s = re.sub(r"<[^>]+>", "", s or "")
    return re.sub(r"\s+", " ", s).replace("&amp;", "&").replace("&quot;", '"').strip()


def fetch_threads(con, appid, pages=4, verbose=True, debug_dir=None):
    now, today = int(time.time()), datetime.now(TW).strftime("%Y-%m-%d")
    rows, first_html = [], None
    for p in range(1, pages + 1):
        url = f"https://steamcommunity.com/app/{appid}/discussions/0/?fp={p}"
        try:
            html = http_get(url)
        except Exception as e:
            print(f"  ! 討論版抓取失敗 p{p}: {e}")
            break
        if p == 1:
            first_html = html
        topics = parse_topics(html, appid)
        rows += [(t[0], appid, t[1], t[2], t[3], t[4], now) for t in topics]
        if verbose:
            print(f"  [{appid}] 討論版第 {p} 頁：{len(topics)} 串")
        if not topics:
            break
        time.sleep(1.0)

    if not rows and first_html is not None and debug_dir:
        # 一串都沒解析到 → 把現場留下來，下次好對照（Steam 改版／被擋都看得出來）
        try:
            os.makedirs(debug_dir, exist_ok=True)
            with open(os.path.join(debug_dir, "_threads_debug.txt"), "w",
                      encoding="utf-8") as f:
                f.write(f"# {datetime.now(TW):%Y-%m-%d %H:%M} appid={appid}\n"
                        f"# html_len={len(first_html)} "
                        f"forum_topic={first_html.count('forum_topic')} "
                        f"discussions_href={first_html.count('/discussions/')}\n\n")
                f.write(first_html[:6000])
        except Exception:
            pass

    prev = {r["id"]: r["replies"] for r in
            con.execute("SELECT id, replies FROM threads WHERE appid=?", (appid,))}
    changed = []
    for tid, aid, title, author, replies, url, seen in rows:
        old = prev.get(tid)
        kind = delta = None
        if old is None:
            kind, delta = "new", replies
            changed.append(("new", tid, title, author, replies, url))
        elif replies > old:
            kind, delta = "reply", replies - old
            changed.append(("reply", tid, title, author, replies - old, url))
        prev[tid] = replies
        con.execute(
            """INSERT INTO threads (id,appid,title,author,replies,url,first_seen,last_seen)
               VALUES (?,?,?,?,?,?,?,?)
               ON CONFLICT(id) DO UPDATE SET title=excluded.title, replies=excluded.replies,
                 author=excluded.author, url=excluded.url, last_seen=excluded.last_seen""",
            (tid, aid, title, author, replies, url, seen, seen))
        if kind:
            con.execute(
                """INSERT INTO thread_activity (tid,day,kind,delta,replies)
                   VALUES (?,?,?,?,?)
                   ON CONFLICT(tid,day) DO UPDATE SET
                     delta = thread_activity.delta + excluded.delta,
                     replies = excluded.replies,
                     kind = CASE WHEN thread_activity.kind='new' THEN 'new'
                                 ELSE excluded.kind END""",
                (tid, today, kind, delta, replies))
    con.commit()
    return changed


# ------------------------------------------------------------ 分析 / 報表 ----

_KW_CACHE = {}


def _kw_match(kw, text):
    """CJK 關鍵詞用子字串比對；純英數關鍵詞加詞界，避免 sl 命中 slob、ui 命中 build。"""
    if any("⺀" <= ch <= "鿿" or "가" <= ch <= "힯" or
           "぀" <= ch <= "ヿ" for ch in kw):
        return kw in text
    pat = _KW_CACHE.get(kw)
    if pat is None:
        pat = _KW_CACHE[kw] = re.compile(
            r"(?<![0-9a-z])" + re.escape(kw) + r"s?(?![0-9a-z])")
    return bool(pat.search(text))


def tag_themes(text):
    t = (text or "").lower()
    return [name for name, kws in THEMES if any(_kw_match(k, t) for k in kws)]


def group_by_theme(rows, threads, days=None):
    """把評論與討論串依議題歸類；同一筆若命中多個議題會出現在多組（原文只有一份）。"""
    cutoff = int(time.time()) - days * 86400 if days else 0
    groups = {name: {"reviews": [], "threads": [], "pos": 0, "neg": 0} for name, _ in THEMES}
    groups["其他 / 未分類"] = {"reviews": [], "threads": [], "pos": 0, "neg": 0}
    for r in rows:
        if r["ts_created"] < cutoff:
            continue
        names = tag_themes(r["text"]) or ["其他 / 未分類"]
        for n in names:
            g = groups[n]
            g["reviews"].append(r)
            g["pos" if r["voted_up"] else "neg"] += 1
    for t in threads or []:
        for n in (tag_themes(t.get("title")) or ["其他 / 未分類"]):
            groups[n]["threads"].append(t)
    for g in groups.values():
        # 負評優先、再按新到舊；討論串按回覆數
        g["reviews"].sort(key=lambda r: (r["voted_up"], -r["ts_created"]))
        g["threads"].sort(key=lambda t: -(t.get("replies") or 0))
    order = [n for n, _ in THEMES] + ["其他 / 未分類"]
    return [(n, groups[n]) for n in order
            if groups[n]["reviews"] or groups[n]["threads"]]


def analyse(con, appid, days=30, since=None):
    since_ts = parse_since(since if since is not None else SINCE_DEFAULT)
    rows = [dict(r) for r in con.execute(
        "SELECT * FROM reviews WHERE appid=? AND ts_created>=? ORDER BY ts_created DESC",
        (appid, since_ts))]
    cutoff = int(time.time()) - days * 86400
    latest_sum = {}
    for r in con.execute(
            """SELECT * FROM summaries WHERE appid=? AND checked_at=
                 (SELECT MAX(checked_at) FROM summaries WHERE appid=?)""", (appid, appid)):
        latest_sum[r["lang"]] = dict(r)
    prev_sum = {}
    for r in con.execute(
            """SELECT * FROM summaries WHERE appid=? AND checked_at=
                 (SELECT MAX(checked_at) FROM summaries WHERE appid=? AND checked_at <
                   (SELECT MAX(checked_at) FROM summaries WHERE appid=?))""",
            (appid, appid, appid)):
        prev_sum[r["lang"]] = dict(r)

    # 每日走勢
    daily = {}
    for r in rows:
        d = datetime.fromtimestamp(r["ts_created"], TW).strftime("%Y-%m-%d")
        b = daily.setdefault(d, {"pos": 0, "neg": 0})
        b["pos" if r["voted_up"] else "neg"] += 1

    # 主題統計（近 days 天）
    theme_stat = {name: {"pos": 0, "neg": 0, "ids": []} for name, _ in THEMES}
    for r in rows:
        if r["ts_created"] < cutoff:
            continue
        for name in tag_themes(r["text"]):
            theme_stat[name]["pos" if r["voted_up"] else "neg"] += 1
            theme_stat[name]["ids"].append(r["id"])

    # 語言分佈（樣本）
    lang_stat = {}
    for r in rows:
        b = lang_stat.setdefault(r["lang"], {"pos": 0, "neg": 0})
        b["pos" if r["voted_up"] else "neg"] += 1

    return {"rows": rows, "summary": latest_sum, "prev_summary": prev_sum,
            "daily": dict(sorted(daily.items())), "themes": theme_stat,
            "langs": lang_stat, "days": days}


def digest_text(con, appid, new_reviews, thread_changes, a, threads=None, excerpts=6):
    s = a["summary"].get("all", {})
    p = a["prev_summary"].get("all", {})
    L = [f"# {APPS.get(appid, appid)} — {datetime.now(TW):%Y-%m-%d %H:%M} 監控摘要", ""]
    if s:
        rate = 100 * s["positive"] / s["total"] if s["total"] else 0
        line = (f"- 總評論 **{s['total']}**（好評 {s['positive']} / 負評 {s['negative']}"
                f" = 好評率 **{rate:.1f}%**，Steam 評價：{s['score_desc']}）")
        if p and p.get("total") is not None:
            d_t, d_n = s["total"] - p["total"], s["negative"] - p["negative"]
            line += f"　↔ 上次：評論 {d_t:+d}、負評 {d_n:+d}"
        L.append(line)
    for lang in DEFAULT_LANGS:
        ls = a["summary"].get(lang)
        if ls and ls["total"]:
            L.append(f"    - {LANG_LABEL.get(lang, lang)}：{ls['total']} 筆，"
                     f"好評率 {100*ls['positive']/ls['total']:.0f}%（{ls['score_desc']}）")
    L += ["", f"## 新評論 {len(new_reviews)} 筆"]
    negs = [r for r in new_reviews if not r[3]]
    if negs:
        L.append(f"其中 **負評 {len(negs)} 筆**：")
        for r in negs[:10]:
            L.append(f"  - [{LANG_LABEL.get(r[2], r[2])}] {r[13][:120].replace(chr(10),' ')}")
    if not new_reviews:
        L.append("（無新評論）")
    groups = group_by_theme(a["rows"], threads or [], days=a["days"])
    groups.sort(key=lambda kv: (kv[0] == "其他 / 未分類",
                                -(len(kv[1]["reviews"]) + len(kv[1]["threads"]))))
    if groups:
        L += ["", f"## 議題彙整（近 {a['days']} 天）",
              "同一議題的評論與討論串放在一起；一筆評論談到多件事會出現在多個議題。"]
        for n, g in groups:
            L += ["", f"### {n}　評論 {len(g['reviews'])} 筆"
                      f"（好評 {g['pos']} / 負評 {g['neg']}）"
                      + (f"、討論串 {len(g['threads'])} 串" if g["threads"] else "")]
            for t in g["threads"][:5]:
                L.append(f"  - 💬 {t['title']}（{t['author']} · {t['replies']} 回覆）{t['url']}")
            for r in g["reviews"][:excerpts]:
                mark = "👍" if r["voted_up"] else "👎"
                txt = " ".join((r["text"] or "").split())[:160]
                L.append(f"  - {mark} [{LANG_LABEL.get(r['lang'], r['lang'])}] {txt}")
            if len(g["reviews"]) > excerpts:
                L.append(f"  - …另有 {len(g['reviews']) - excerpts} 筆，見 dashboard")
    if thread_changes:
        L += ["", "## 討論版異動"]
        for kind, tid, title, author, n, url in thread_changes[:20]:
            tag = "🆕 新開串" if kind == "new" else f"💬 +{n} 回覆"
            L.append(f"  - {tag}｜{title}（{author}）{url}")
    return "\n".join(L)


# ------------------------------------------------------------- HTML 儀表板 ---

PALETTE = {  # dataviz 參考色盤
    "pos_l": "#2a78d6", "pos_d": "#3987e5",
    "neg_l": "#d03b3b", "neg_d": "#e66767",
}


# 只採計這個日期（含）之後的評論與討論串。
# 正式版 2026-09-09 上線，之前的都是試玩版玩家的回饋，混進來會誤導 BUG 判讀。
SINCE_DEFAULT = "2026-09-09"


def parse_since(s):
    """'2026-09-09' → unix ts（台北時區當日 00:00）。空字串或 None 表示不設限。"""
    if not s:
        return 0
    return int(datetime.strptime(s, "%Y-%m-%d").replace(tzinfo=TW).timestamp())


def esc(s):
    return (str(s or "").replace("&", "&amp;").replace("<", "&lt;").replace(">", "&gt;"))


def review_card(r):
    pill = '<span class="pill up">👍 好評</span>' if r["voted_up"] \
        else '<span class="pill dn">👎 負評</span>'
    useful = f'<span>有用 {r["votes_up"]}</span>' if r["votes_up"] else ""
    day = datetime.fromtimestamp(r["ts_created"], TW).strftime("%Y-%m-%d")
    return (f'<div class="rev"><div class="meta">{pill}'
            f'<span>{LANG_LABEL.get(r["lang"], r["lang"])}</span><span>{day}</span>'
            f'<span>遊玩 {round((r["playtime"] or 0)/60, 1)} 小時</span>{useful}'
            f'</div><p>{esc(r["text"])}</p></div>')


def build_html(appid, a, fragment=False, thread_rows=None, note=""):
    s = a["summary"].get("all", {}) or {}
    total = s.get("total", 0) or sum(v["pos"] + v["neg"] for v in a["langs"].values())
    pos = s.get("positive", 0) or sum(v["pos"] for v in a["langs"].values())
    neg = s.get("negative", max(0, total - pos))
    rate = 100 * pos / total if total else 0
    rows = a["rows"]
    gen = datetime.now(TW).strftime("%Y-%m-%d %H:%M")

    # ---- 每日走勢 SVG（堆疊柱）
    daily = a["daily"]
    dmax = max([v["pos"] + v["neg"] for v in daily.values()] or [1])
    W, H, PAD = 880, 220, 28
    n = max(len(daily), 1)
    bw = max(3.0, min(22.0, (W - 2 * PAD) / n - 3))
    step = (W - 2 * PAD) / n
    bars, xlabels = [], []
    for i, (d, v) in enumerate(daily.items()):
        x = PAD + i * step + (step - bw) / 2
        tot = v["pos"] + v["neg"]
        hp = (H - 2 * PAD) * v["pos"] / dmax
        hn = (H - 2 * PAD) * v["neg"] / dmax
        y0 = H - PAD
        if v["neg"]:
            bars.append(f'<rect class="b-neg" x="{x:.1f}" y="{y0-hn:.1f}" width="{bw:.1f}" '
                        f'height="{hn:.1f}" rx="2"><title>{d}　負評 {v["neg"]}</title></rect>')
        if v["pos"]:
            bars.append(f'<rect class="b-pos" x="{x:.1f}" y="{y0-hn-hp-(2 if v["neg"] else 0):.1f}" '
                        f'width="{bw:.1f}" height="{hp:.1f}" rx="2">'
                        f'<title>{d}　好評 {v["pos"]}</title></rect>')
        if n <= 14 or i % max(1, n // 8) == 0:
            xlabels.append(f'<text class="ax" x="{x+bw/2:.1f}" y="{H-8}" text-anchor="middle">'
                           f'{d[5:]}</text>')
    grid = "".join(
        f'<line class="gr" x1="{PAD}" x2="{W-PAD}" y1="{H-PAD-(H-2*PAD)*k/2:.1f}" '
        f'y2="{H-PAD-(H-2*PAD)*k/2:.1f}"/>'
        f'<text class="ax" x="{PAD-6}" y="{H-PAD-(H-2*PAD)*k/2+4:.1f}" text-anchor="end">'
        f'{dmax*k/2:.0f}</text>' for k in (0, 1, 2))
    daily_svg = (f'<svg viewBox="0 0 {W} {H}" role="img" aria-label="每日評論數">'
                 f'{grid}{"".join(bars)}{"".join(xlabels)}</svg>')

    # ---- 語言好評率（官方摘要優先）
    lang_bars = []
    lang_src = [(l, a["summary"][l]) for l in DEFAULT_LANGS
                if a["summary"].get(l) and a["summary"][l]["total"]]
    if not lang_src:
        lang_src = [(l, {"total": v["pos"] + v["neg"], "positive": v["pos"],
                         "negative": v["neg"], "score_desc": ""})
                    for l, v in a["langs"].items()]
    lang_src.sort(key=lambda kv: -kv[1]["total"])
    lmax = max([v["total"] for _, v in lang_src] or [1])
    for lang, v in lang_src:
        r = 100 * v["positive"] / v["total"]
        wp = 100 * v["positive"] / lmax
        wn = 100 * v["negative"] / lmax
        lang_bars.append(
            f'<div class="lrow"><div class="llab">{LANG_LABEL.get(lang, lang)}</div>'
            f'<div class="lbar"><span class="s-pos" style="width:{wp:.2f}%" '
            f'title="好評 {v["positive"]}"></span>'
            f'<span class="s-neg" style="width:{wn:.2f}%" title="負評 {v["negative"]}"></span></div>'
            f'<div class="lval"><b>{r:.0f}%</b> <span class="mut">/ {v["total"]} 筆</span></div></div>')

    # ---- 議題彙整：評論 + 討論串併在同一組
    groups = group_by_theme(rows, thread_rows or [], days=a["days"])
    groups.sort(key=lambda kv: (kv[0] == "其他 / 未分類",
                                -(len(kv[1]["reviews"]) + len(kv[1]["threads"]))))
    gmax = max([len(g["reviews"]) for _, g in groups] or [1])
    theme_blocks = []
    for name, g in groups:
        nr, nt = len(g["reviews"]), len(g["threads"])
        wp = 100 * g["pos"] / gmax if gmax else 0
        wn = 100 * g["neg"] / gmax if gmax else 0
        tl = "".join(
            f'<li><a href="{t["url"]}" target="_blank" rel="noopener">{esc(t["title"])}</a>'
            f'<span class="mut"> · {esc(t["author"] or "")} · {t["replies"]} 回覆</span></li>'
            for t in g["threads"])
        rl = "".join(review_card(r) for r in g["reviews"])
        theme_blocks.append(
            f'<details class="grp"><summary>'
            f'<span class="gname">{name}</span>'
            f'<span class="gbar"><span class="s-pos" style="width:{wp:.2f}%"></span>'
            f'<span class="s-neg" style="width:{wn:.2f}%"></span></span>'
            f'<span class="gnum">評論 <b>{nr}</b>'
            f'<span class="mut">（好 {g["pos"]}／負 {g["neg"]}）</span>'
            f'{f"　討論串 <b>{nt}</b>" if nt else ""}</span></summary>'
            f'<div class="gbody">'
            f'{f"<div class=gsec><div class=gsech>相關討論串</div><ol class=threads>{tl}</ol></div>" if tl else ""}'
            f'{f"<div class=gsec><div class=gsech>相關評論原文</div>{rl}</div>" if rl else ""}'
            f'</div></details>')
    theme_rows = theme_blocks

    # ---- 評論資料（給前端篩選用）
    data = [{"id": r["id"], "lang": r["lang"], "up": bool(r["voted_up"]),
             "ts": r["ts_created"], "pt": r["playtime"], "vu": r["votes_up"],
             "txt": r["text"], "th": tag_themes(r["text"])} for r in rows]
    langs_present = sorted({r["lang"] for r in rows})
    lang_opts = "".join(f'<option value="{l}">{LANG_LABEL.get(l, l)}</option>' for l in langs_present)
    theme_opts = "".join(f'<option value="{n}">{n}</option>' for n, _ in THEMES)

    prev = a["prev_summary"].get("all")
    delta = ""
    if prev and prev.get("total") is not None and s:
        dt = s["total"] - prev["total"]
        dn = s["negative"] - prev["negative"]
        delta = (f'<div class="delta">與上次相比：評論 <b>{dt:+d}</b>　負評 <b>{dn:+d}</b></div>')

    thread_html = ""
    if thread_rows:
        items = "".join(
            f'<li><a href="{t["url"]}" target="_blank" rel="noopener">{t["title"]}</a>'
            f'<span class="mut"> · {t["author"]} · {t["replies"]} 回覆</span></li>'
            for t in sorted(thread_rows, key=lambda t: -(t.get("replies") or 0))[:12])
        thread_html = (f'<section class="card"><h2>討論版熱門串（依回覆數）</h2>'
                       f'<ol class="threads">{items}</ol></section>')

    css = """
<style>
.viz-root{color-scheme:light;--surface-1:#fcfcfb;--plane:#f9f9f7;--ink:#0b0b0b;
 --ink2:#52514e;--mut:#898781;--grid:#e1e0d9;--line:#c3c2b7;
 --pos:#2a78d6;--neg:#d03b3b;--ring:rgba(11,11,11,.10);
 font:14px/1.55 system-ui,-apple-system,"Segoe UI","Noto Sans TC",sans-serif;
 color:var(--ink);background:var(--plane);padding:20px;max-width:1080px;margin:0 auto}
@media (prefers-color-scheme:dark){:root:where(:not([data-theme=light])) .viz-root{
 color-scheme:dark;--surface-1:#1a1a19;--plane:#0d0d0d;--ink:#fff;--ink2:#c3c2b7;
 --grid:#2c2c2a;--line:#383835;--pos:#3987e5;--neg:#e66767;--ring:rgba(255,255,255,.10)}}
:root[data-theme=dark] .viz-root{color-scheme:dark;--surface-1:#1a1a19;--plane:#0d0d0d;
 --ink:#fff;--ink2:#c3c2b7;--grid:#2c2c2a;--line:#383835;--pos:#3987e5;--neg:#e66767;
 --ring:rgba(255,255,255,.10)}
.viz-root h1{font-size:20px;margin:0 0 2px}
.viz-root h2{font-size:15px;margin:0 0 14px;color:var(--ink2);font-weight:600}
.sub{color:var(--mut);font-size:12.5px;margin-bottom:18px}
.card{background:var(--surface-1);border:1px solid var(--ring);border-radius:12px;
 padding:18px 20px;margin-bottom:16px}
.tiles{display:grid;grid-template-columns:repeat(auto-fit,minmax(150px,1fr));gap:12px;margin-bottom:16px}
.tile{background:var(--surface-1);border:1px solid var(--ring);border-radius:12px;padding:14px 16px}
.tile .k{font-size:11.5px;color:var(--mut);letter-spacing:.02em}
.tile .v{font-size:26px;font-weight:650;margin-top:2px}
.tile .n{font-size:12px;color:var(--ink2)}
.delta{font-size:12.5px;color:var(--ink2);margin-top:6px}
svg{width:100%;height:auto;display:block}
.gr{stroke:var(--grid);stroke-width:1}
.ax{fill:var(--mut);font-size:10px}
.b-pos{fill:var(--pos)}.b-neg{fill:var(--neg)}
.legend{display:flex;gap:16px;font-size:12.5px;color:var(--ink2);margin:10px 0 0}
.legend i{display:inline-block;width:10px;height:10px;border-radius:2px;margin-right:5px}
.lrow{display:grid;grid-template-columns:112px 1fr 118px;gap:10px;align-items:center;margin:7px 0}
.llab{font-size:12.5px;color:var(--ink2);text-align:right}
.lbar{display:flex;gap:2px;height:14px;background:transparent}
.lbar span{display:block;border-radius:0 3px 3px 0}
.s-pos{background:var(--pos)}.s-neg{background:var(--neg)}
.lval{font-size:12.5px;font-variant-numeric:tabular-nums}
.mut{color:var(--mut)}
.filters{display:flex;flex-wrap:wrap;gap:8px;margin-bottom:12px}
.filters select,.filters input{font:inherit;font-size:13px;padding:6px 9px;border-radius:8px;
 border:1px solid var(--line);background:var(--surface-1);color:var(--ink)}
.rev{border-top:1px solid var(--grid);padding:11px 0}
.rev .meta{font-size:11.5px;color:var(--mut);display:flex;gap:9px;flex-wrap:wrap;align-items:center}
.pill{font-size:11px;padding:1px 7px;border-radius:99px;border:1px solid var(--ring)}
.pill.up{color:var(--pos)}.pill.dn{color:var(--neg)}
.tg{font-size:11px;color:var(--ink2);background:var(--plane);border:1px solid var(--ring);
 padding:1px 6px;border-radius:6px}
.rev p{margin:6px 0 0;white-space:pre-wrap;word-break:break-word}
.threads{margin:0;padding-left:20px}.threads li{margin:5px 0}
.threads a{color:var(--pos)}
.foot{color:var(--mut);font-size:11.5px;margin-top:18px}
table.tv{border-collapse:collapse;font-size:12.5px;width:100%;margin-top:8px}
table.tv th,table.tv td{border-bottom:1px solid var(--grid);padding:5px 8px;text-align:right}
table.tv th:first-child,table.tv td:first-child{text-align:left}
details summary{cursor:pointer;color:var(--ink2);font-size:12.5px}
.grp{border-top:1px solid var(--grid)}
.grp:last-of-type{border-bottom:1px solid var(--grid)}
.grp>summary{display:grid;grid-template-columns:150px 1fr 250px;gap:12px;align-items:center;
 padding:9px 4px;color:var(--ink);font-size:13px;list-style:none}
.grp>summary::-webkit-details-marker{display:none}
.grp>summary:hover{background:var(--plane)}
.grp>summary:focus-visible{outline:2px solid var(--pos);outline-offset:2px}
.grp[open]>summary{font-weight:600}
.gname::before{content:"▸ ";color:var(--mut)}
.grp[open] .gname::before{content:"▾ "}
.gbar{display:flex;gap:2px;height:12px}
.gbar span{display:block;border-radius:0 3px 3px 0}
.gnum{font-size:12.5px;color:var(--ink2);font-variant-numeric:tabular-nums;text-align:right}
.gbody{padding:4px 0 16px 14px;border-left:2px solid var(--grid);margin-left:6px}
.gsec{margin-top:10px}
.gsech{font-size:11.5px;letter-spacing:.04em;color:var(--mut);margin-bottom:4px}
.gbody .rev:first-of-type{border-top:none}
@media (max-width:720px){.grp>summary{grid-template-columns:1fr;gap:4px}
 .gnum{text-align:left}.lrow{grid-template-columns:80px 1fr 96px}}
</style>"""

    table_view = "".join(
        f"<tr><td>{LANG_LABEL.get(l, l)}</td><td>{v['total']}</td><td>{v['positive']}</td>"
        f"<td>{v['negative']}</td><td>{100*v['positive']/v['total']:.1f}%</td></tr>"
        for l, v in lang_src)

    body = f"""
<div class="viz-root" data-palette="#2a78d6,#d03b3b">
<h1>{APPS.get(appid, appid)} · 玩家評論監控</h1>
<div class="sub">產出時間 {gen}（台北）　資料來源：Steam appreviews API (appid {appid}) 與社群討論版{('　· ' + note) if note else ''}</div>

<div class="tiles">
  <div class="tile"><div class="k">總評論數</div><div class="v">{total}</div>
    <div class="n">{s.get('score_desc','')}</div></div>
  <div class="tile"><div class="k">好評率</div><div class="v">{rate:.1f}%</div>
    <div class="n">好評 {pos}／負評 {neg}</div></div>
  <div class="tile"><div class="k">繁中好評率</div><div class="v">{
      (lambda t: f"{100*t['positive']/t['total']:.0f}%" if t and t['total'] else "—")(a["summary"].get("tchinese"))
    }</div><div class="n">{
      (lambda t: f"{t['total']} 筆 · {t['score_desc']}" if t and t['total'] else "尚無資料")(a["summary"].get("tchinese"))
    }</div></div>
  <div class="tile"><div class="k">本地樣本</div><div class="v">{len(rows)}</div>
    <div class="n">已存入資料庫可全文檢索</div></div>
</div>
{delta}

<section class="card">
  <h2>每日評論數（好評 / 負評）</h2>
  {daily_svg}
  <div class="legend"><span><i style="background:var(--pos)"></i>好評</span>
    <span><i style="background:var(--neg)"></i>負評</span></div>
</section>

<section class="card">
  <h2>各語言好評率</h2>
  {''.join(lang_bars)}
  <div class="legend"><span><i style="background:var(--pos)"></i>好評</span>
    <span><i style="background:var(--neg)"></i>負評</span></div>
  <details style="margin-top:12px"><summary>表格檢視</summary>
   <table class="tv"><thead><tr><th>語言</th><th>評論</th><th>好評</th><th>負評</th><th>好評率</th></tr></thead>
   <tbody>{table_view}</tbody></table></details>
</section>

<section class="card">
  <h2>議題彙整 — 同一議題的評論與討論串放在一起（近 {a['days']} 天）</h2>
  <div class="mut" style="font-size:12.5px;margin:-6px 0 12px">
    點開任一議題即可看到該議題下的討論串與評論原文；一筆評論若同時談到多件事，會出現在多個議題裡。</div>
  {''.join(theme_rows) or '<div class="mut">尚無資料</div>'}
  <div class="legend"><span><i style="background:var(--pos)"></i>好評中提及</span>
    <span><i style="background:var(--neg)"></i>負評中提及</span></div>
</section>

{thread_html}

<section class="card">
  <h2>評論瀏覽器</h2>
  <div class="filters">
    <select id="f-lang"><option value="">全部語言</option>{lang_opts}</select>
    <select id="f-up"><option value="">好評+負評</option><option value="1">只看好評</option>
      <option value="0">只看負評</option></select>
    <select id="f-th"><option value="">全部議題</option>{theme_opts}</select>
    <input id="f-q" type="search" placeholder="全文搜尋…" style="flex:1;min-width:160px">
    <select id="f-sort"><option value="new">最新優先</option><option value="vote">最多人覺得有用</option>
      <option value="long">最長優先</option></select>
  </div>
  <div id="count" class="mut" style="font-size:12.5px"></div>
  <div id="list"></div>
</section>

<div class="foot">好評率＝Steam 官方 query_summary；議題統計為關鍵詞比對，僅供快速定位，細節請點開原文確認。</div>
</div>
<script>
const DATA = {json.dumps(data, ensure_ascii=False)};
const LANGL = {json.dumps(LANG_LABEL, ensure_ascii=False)};
const $ = s => document.querySelector(s);
function fmt(ts){{const d=new Date(ts*1000);return d.toLocaleString('zh-TW',{{timeZone:'Asia/Taipei',
  year:'numeric',month:'2-digit',day:'2-digit',hour:'2-digit',minute:'2-digit'}});}}
function render(){{
  const lang=$('#f-lang').value, up=$('#f-up').value, th=$('#f-th').value,
        q=$('#f-q').value.trim().toLowerCase(), sort=$('#f-sort').value;
  let rows=DATA.filter(r=>(!lang||r.lang===lang)&&(up===''||String(r.up?1:0)===up)
        &&(!th||r.th.includes(th))&&(!q||r.txt.toLowerCase().includes(q)));
  rows.sort(sort==='vote'?(a,b)=>b.vu-a.vu||b.ts-a.ts
          :sort==='long'?(a,b)=>b.txt.length-a.txt.length:(a,b)=>b.ts-a.ts);
  $('#count').textContent = rows.length+' 筆（共 '+DATA.length+' 筆樣本）';
  $('#list').innerHTML = rows.slice(0,400).map(r=>`<div class="rev"><div class="meta">
     <span class="pill ${{r.up?'up':'dn'}}">${{r.up?'👍 好評':'👎 負評'}}</span>
     <span>${{LANGL[r.lang]||r.lang}}</span><span>${{fmt(r.ts)}}</span>
     <span>遊玩 ${{Math.round(r.pt/60*10)/10}} 小時</span>
     ${{r.vu?`<span>有用 ${{r.vu}}</span>`:''}}
     ${{r.th.map(t=>`<span class="tg">${{t}}</span>`).join('')}}
   </div><p>${{r.txt.replace(/[<>&]/g,c=>({{'<':'&lt;','>':'&gt;','&':'&amp;'}})[c])}}</p></div>`).join('');
}}
['#f-lang','#f-up','#f-th','#f-q','#f-sort'].forEach(s=>{{
  document.querySelector(s).addEventListener('input',render);}});
render();
</script>"""

    if fragment:
        return f"<title>幻世錄重製版 評論監控</title>{css}{body}"
    return (f'<!doctype html><html lang="zh-Hant"><head><meta charset="utf-8">'
            f'<meta name="viewport" content="width=device-width,initial-scale=1">'
            f'<title>幻世錄重製版 評論監控</title>{css}</head><body style="margin:0">{body}</body></html>')


# ------------------------------------------------------------------ CLI ------

def cmd_fetch(args, con):
    total_new, all_thread_changes = {}, []
    for appid in APPS:
        print(f"→ {APPS[appid]} ({appid})")
        fetch_summaries(con, appid, args.langs.split(","))
        total_new[appid] = fetch_reviews(con, appid, full=args.full)
        print(f"  新增評論 {len(total_new[appid])} 筆")
    if not args.no_threads:
        for appid in DISCUSSION_APPS:
            all_thread_changes += fetch_threads(con, appid, pages=args.thread_pages,
                                                debug_dir=args.out)
    con.execute("INSERT INTO runs (started_at,new_reviews,new_threads,note) VALUES (?,?,?,?)",
                (int(time.time()), sum(len(v) for v in total_new.values()),
                 len(all_thread_changes), "full" if args.full else "incremental"))
    con.commit()
    return total_new, all_thread_changes


def cmd_report(args, con, new_reviews=None, thread_changes=None):
    os.makedirs(args.out, exist_ok=True)
    stamp = datetime.now(TW).strftime("%Y%m%d")
    written = []
    for appid in APPS:
        a = analyse(con, appid, days=args.days, since=args.since)
        if not a["rows"] and not (a["summary"].get("all", {}).get("total")):
            print(f"→ 略過 {APPS.get(appid, appid)}：目前沒有評論資料")
            continue
        # 討論串同樣只採計上線後首次出現的（first_seen 當作發文時間的代理值）
        threads = [dict(r) for r in con.execute(
            "SELECT * FROM threads WHERE first_seen>=? ORDER BY replies DESC, last_seen DESC",
            (parse_since(args.since),))]
        html = build_html(appid, a, fragment=args.fragment, thread_rows=threads, note=args.note)
        hp = os.path.join(args.out, f"dashboard_{appid}.html")
        open(hp, "w", encoding="utf-8").write(html)
        md = digest_text(con, appid, (new_reviews or {}).get(appid, []),
                         thread_changes or [], a, threads=threads, excerpts=args.excerpts)
        mp = os.path.join(args.out, f"report_{appid}_{stamp}.md")
        open(mp, "w", encoding="utf-8").write(md)
        written += [hp, mp]
        print("\n" + md + "\n")
        if args.webhook:
            try:
                payload = json.dumps({"content": md[:1900], "text": md[:1900]}).encode()
                req = urllib.request.Request(args.webhook, data=payload,
                                             headers={"Content-Type": "application/json"})
                urllib.request.urlopen(req, timeout=20)
                print("  已推送 webhook")
            except Exception as e:
                print(f"  ! webhook 失敗: {e}")
    print("輸出：" + ", ".join(written))


def cmd_seed(args, con):
    """把 JSONL（id,lang,up,ts,pt,vu,vf,cc,sp,rf,txt）匯入，方便離線測試 / 手動補資料。"""
    now, rows = int(time.time()), []
    for line in open(args.file, encoding="utf-8"):
        line = line.strip()
        if not line:
            continue
        r = json.loads(line)
        rows.append((str(r["id"]), args.appid, r.get("lang"), 1 if r.get("up") else 0,
                     r["ts"], r.get("ts", 0), r.get("pt", 0), r.get("vu", 0), r.get("vf", 0),
                     r.get("cc", 0), 1 if r.get("sp") else 0, 1 if r.get("rf") else 0, 0,
                     r.get("txt", ""), None, now))
    con.executemany("INSERT OR IGNORE INTO reviews VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)", rows)
    con.commit()
    print(f"匯入 {len(rows)} 筆到 appid {args.appid}")


def cmd_export(args, con):
    """把抓到的評論匯出成 JSONL（欄位與 seed 相同），方便丟回對話裡做議題彙整。"""
    rows = con.execute(
        "SELECT * FROM reviews WHERE appid=? ORDER BY ts_created DESC", (args.appid,))
    out = args.file or f"reviews_{args.appid}.jsonl"
    n = 0
    with open(out, "w", encoding="utf-8") as f:
        for r in rows:
            f.write(json.dumps({
                "id": r["id"], "lang": r["lang"], "up": bool(r["voted_up"]),
                "ts": r["ts_created"], "pt": r["playtime"], "vu": r["votes_up"],
                "vf": r["votes_funny"], "cc": r["comment_count"],
                "sp": bool(r["steam_purchase"]), "rf": bool(r["refunded"]),
                "txt": r["text"],
            }, ensure_ascii=False) + "\n")
            n += 1
    print(f"已匯出 {n} 筆到 {out}　← 把這個檔案丟回 Claude 對話裡即可")


def cmd_export_threads(args, con):
    """討論串匯出成 JSON：一串在哪一天有動靜（新開／有新回覆），就算那天的內容。

    同一串在 9/12 開、9/14 有人回，就會出現在 9/12 也出現在 9/14，兩筆都能點進原串。
    --state 另外存一份完整狀態，讓下一次執行（GitHub Actions 每次都是新資料庫）
    知道「上次的回覆數是多少」，才分得出來今天多了幾則。
    """
    out = args.file or f"threads_{args.appid}.json"
    meta = {r["id"]: r for r in
            con.execute("SELECT * FROM threads WHERE appid=?", (args.appid,))}
    acts = list(con.execute(
        "SELECT * FROM thread_activity WHERE tid IN "
        "(SELECT id FROM threads WHERE appid=?) ORDER BY day DESC", (args.appid,)))

    rows, covered = [], set()
    for a in acts:
        t = meta.get(a["tid"])
        if not t:
            continue
        covered.add(a["tid"])
        rows.append({
            "id": f'{a["tid"]}@{a["day"]}', "tid": a["tid"], "d": a["day"],
            "t": t["title"], "a": t["author"] or "", "r": a["replies"] or t["replies"] or 0,
            "u": t["url"], "cat": "討論串", "x": "",
            "k": a["kind"], "n": a["delta"] or 0,
        })
    # 還沒有動態紀錄的（舊資料庫匯入的）→ 用首次發現當日期
    for tid, t in meta.items():
        if tid in covered:
            continue
        rows.append({
            "id": tid, "tid": tid,
            "d": datetime.fromtimestamp(t["first_seen"], TW).strftime("%Y-%m-%d"),
            "t": t["title"], "a": t["author"] or "", "r": t["replies"] or 0,
            "u": t["url"], "cat": "討論串", "x": "", "k": "new", "n": 0,
        })
    rows.sort(key=lambda r: (r["d"], r["r"]), reverse=True)
    json.dump(rows, open(out, "w", encoding="utf-8"), ensure_ascii=False)
    print(f"已匯出 {len(rows)} 筆討論串動態（{len(meta)} 串）到 {out}")

    if args.state:
        os.makedirs(os.path.dirname(os.path.abspath(args.state)) or ".", exist_ok=True)
        with open(args.state, "w", encoding="utf-8") as f:
            for tid, t in meta.items():
                f.write(json.dumps({
                    "id": tid, "appid": t["appid"], "t": t["title"], "a": t["author"],
                    "r": t["replies"], "u": t["url"],
                    "fs": t["first_seen"], "ls": t["last_seen"],
                    "act": [[x["day"], x["kind"], x["delta"], x["replies"]]
                            for x in acts if x["tid"] == tid],
                }, ensure_ascii=False) + "\n")
        print(f"已寫入討論串狀態 {args.state}")


def cmd_seed_threads(args, con):
    """把上一次執行存下來的討論串狀態讀回來（GitHub Actions 每次都是空資料庫）。"""
    path = args.file or args.state
    if not path or not os.path.exists(path):
        print(f"沒有討論串狀態檔可讀：{path}")
        return
    n = a = 0
    for line in open(path, encoding="utf-8"):
        line = line.strip()
        if not line:
            continue
        d = json.loads(line)
        con.execute(
            """INSERT INTO threads (id,appid,title,author,replies,url,first_seen,last_seen)
               VALUES (?,?,?,?,?,?,?,?) ON CONFLICT(id) DO NOTHING""",
            (d["id"], d.get("appid", args.appid), d.get("t", ""), d.get("a", ""),
             d.get("r", 0), d.get("u", ""), d.get("fs", 0), d.get("ls", 0)))
        n += 1
        for row in d.get("act", []):
            con.execute(
                "INSERT INTO thread_activity (tid,day,kind,delta,replies) VALUES (?,?,?,?,?) "
                "ON CONFLICT(tid,day) DO NOTHING", (d["id"], row[0], row[1], row[2], row[3]))
            a += 1
    con.commit()
    print(f"已讀回 {n} 個討論串、{a} 筆動態")


def cmd_snapshot(args, con):
    """手動寫入官方摘要（沒有網路時，可用 Claude 抓到的數字補）。格式：lang,total,pos,neg,desc"""
    now = int(time.time())
    for item in args.values:
        lang, total, pos, neg, desc = item.split(",", 4)
        con.execute("INSERT OR REPLACE INTO summaries VALUES (?,?,?,?,?,?,?,?)",
                    (args.appid, lang, now, int(total), int(pos), int(neg), 0, desc))
    con.commit()
    print(f"已寫入 {len(args.values)} 筆摘要")


def main():
    p = argparse.ArgumentParser(description="Steam 評論/討論版監控")
    p.add_argument("cmd", choices=["init", "fetch", "report", "run", "seed", "snapshot",
                                   "export", "export-threads", "seed-threads"])
    p.add_argument("file", nargs="?", help="seed 讀取／export 輸出的 JSONL 檔")
    p.add_argument("--appid", type=int, default=4030150)
    p.add_argument("--values", nargs="*", default=[], help="snapshot: lang,total,pos,neg,desc")
    p.add_argument("--db", default="steam_watch.db")
    p.add_argument("--out", default="reports")
    p.add_argument("--full", action="store_true", help="抓全部歷史（第一次跑）")
    p.add_argument("--days", type=int, default=30, help="議題統計視窗")
    p.add_argument("--since", default=SINCE_DEFAULT,
                   help=f"只採計此日期(含)之後的評論與討論串，預設 {SINCE_DEFAULT}（正式版上線日）；"
                        "傳空字串代表不設限")
    p.add_argument("--excerpts", type=int, default=6, help="md 報告每個議題列幾筆評論摘錄")
    p.add_argument("--langs", default=",".join(DEFAULT_LANGS))
    p.add_argument("--thread-pages", type=int, default=4)
    p.add_argument("--no-threads", action="store_true")
    p.add_argument("--webhook", default=os.environ.get("STEAM_WATCH_WEBHOOK"))
    p.add_argument("--fragment", action="store_true", help="輸出 HTML 片段（給 Claude Artifact 用）")
    p.add_argument("--note", default="")
    p.add_argument("--state", default="", help="討論串狀態檔（export-threads 寫出／seed-threads 讀入）")
    args = p.parse_args()

    con = open_db(args.db)
    if args.cmd == "init":
        print(f"資料庫就緒：{args.db}")
    elif args.cmd == "fetch":
        cmd_fetch(args, con)
    elif args.cmd == "report":
        cmd_report(args, con)
    elif args.cmd == "run":
        nr, tc = cmd_fetch(args, con)
        cmd_report(args, con, nr, tc)
    elif args.cmd == "seed":
        cmd_seed(args, con)
    elif args.cmd == "snapshot":
        cmd_snapshot(args, con)
    elif args.cmd == "export":
        cmd_export(args, con)
    elif args.cmd == "export-threads":
        cmd_export_threads(args, con)
    elif args.cmd == "seed-threads":
        cmd_seed_threads(args, con)


if __name__ == "__main__":
    main()
