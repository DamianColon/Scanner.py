# Scanner.py
Stock scanner
import streamlit as st
import yfinance as yf
import pandas as pd
import smtplib
import requests
import time
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from datetime import datetime

# ─── Page Config ──────────────────────────────────────────────────────────────
st.set_page_config(
    page_title="Pre-Market Scanner",
    page_icon="📡",
    layout="wide",
    initial_sidebar_state="expanded",
)

# ─── CSS ──────────────────────────────────────────────────────────────────────
st.markdown("""
<style>
    @import url('https://fonts.googleapis.com/css2?family=IBM+Plex+Mono:wght@400;600;700&display=swap');
    html, body, [class*="css"] { font-family: 'IBM Plex Mono', monospace; background-color: #080b14; color: #d0daf0; }
    .main { background-color: #080b14; }
    .block-container { padding-top: 1.5rem; padding-bottom: 1rem; }
    [data-testid="stSidebar"] { background-color: #0c1020; border-right: 1px solid #141d30; }
    [data-testid="stSidebar"] * { font-family: 'IBM Plex Mono', monospace !important; }
    [data-testid="stMetric"] { background: #0f1622; border: 1px solid #1a2438; border-radius: 8px; padding: 12px 16px; }
    [data-testid="stMetricLabel"] { font-size: 10px !important; letter-spacing: 2px; color: #3a4a65 !important; }
    [data-testid="stMetricValue"] { font-size: 22px !important; font-weight: 700 !important; }
    [data-testid="stDataFrame"] { border: 1px solid #1a2438; border-radius: 8px; overflow: hidden; }
    .stButton button {
        background: linear-gradient(135deg, #1a3060, #0d1e40) !important;
        border: 1px solid #2a4a80 !important; color: #7ab0f0 !important;
        font-family: 'IBM Plex Mono', monospace !important; font-weight: 700 !important;
        letter-spacing: 2px !important; border-radius: 7px !important;
    }
    .stButton button:hover { border-color: #4a7adf !important; color: #aad0ff !important; }
    h1, h2, h3 { font-family: 'IBM Plex Mono', monospace !important; }
    .live-dot {
        display: inline-block; width: 8px; height: 8px; border-radius: 50%;
        background: #3ddf8a; box-shadow: 0 0 10px #3ddf8a; margin-right: 8px;
        animation: blink 1.6s infinite;
    }
    @keyframes blink { 0%,100%{opacity:1} 50%{opacity:0.2} }
    hr { border-color: #141d30 !important; }
    .alert-box {
        background: #0e1a10; border: 1px solid #1a4a20; border-radius: 8px;
        padding: 14px 18px; margin: 6px 0; font-size: 13px;
    }
    .alert-box-warn {
        background: #1a1408; border: 1px solid #4a3a08; border-radius: 8px;
        padding: 14px 18px; margin: 6px 0; font-size: 13px;
    }
</style>
""", unsafe_allow_html=True)

# ─── Session State ─────────────────────────────────────────────────────────────
if "alerted_tickers" not in st.session_state:
    st.session_state.alerted_tickers = set()   # Tracks who we've already alerted this session
if "alert_log" not in st.session_state:
    st.session_state.alert_log = []             # History of alerts sent

# ─── Watchlist ─────────────────────────────────────────────────────────────────
DEFAULT_TICKERS = [
    "NVDA","TSLA","AAPL","AMZN","META","PLTR","SOFI","RIVN",
    "AMD","MSFT","GME","AMC","HOOD","MARA","COIN","SPY","QQQ"
]

# ─── Alert Functions ───────────────────────────────────────────────────────────

def send_push_notification(topic: str, ticker: str, adv_pct: float, change: float, price: float, threshold: float):
    """Send a push notification via ntfy.sh (free, no account needed)."""
    try:
        title   = f"🚨 {ticker} — Pre-Market Alert"
        message = (
            f"{ticker} is trading {adv_pct:.1f}% of its ADV pre-market\n"
            f"Price: ${price:.2f}  |  Change: {'+' if change >= 0 else ''}{change:.2f}%\n"
            f"Threshold: {threshold}% of ADV"
        )
        response = requests.post(
            f"https://ntfy.sh/{topic}",
            data=message.encode("utf-8"),
            headers={
                "Title":    title,
                "Priority": "high" if adv_pct >= 80 else "default",
                "Tags":     "chart_increasing,rotating_light",
            },
            timeout=5,
        )
        return response.status_code == 200
    except Exception as e:
        return False


def send_email_alert(
    smtp_email: str, smtp_password: str, to_email: str,
    ticker: str, adv_pct: float, change: float, price: float, threshold: float
):
    """Send an email alert via Gmail SMTP."""
    try:
        subject = f"🚨 Pre-Market Alert: {ticker} — {adv_pct:.1f}% of ADV"
        body = f"""
Pre-Market Scanner Alert
━━━━━━━━━━━━━━━━━━━━━━━━

Ticker:     {ticker}
% of ADV:   {adv_pct:.1f}%  (threshold: {threshold}%)
Price:      ${price:.2f}
Change:     {'+' if change >= 0 else ''}{change:.2f}%

This stock is trading {adv_pct:.1f}% of its average daily volume
before the market has even opened.

━━━━━━━━━━━━━━━━━━━━━━━━
Sent by Pre-Market Scanner · {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
        """
        msg = MIMEMultipart()
        msg["From"]    = smtp_email
        msg["To"]      = to_email
        msg["Subject"] = subject
        msg.attach(MIMEText(body, "plain"))

        with smtplib.SMTP_SSL("smtp.gmail.com", 465) as server:
            server.login(smtp_email, smtp_password)
            server.send_message(msg)
        return True
    except Exception as e:
        return False


def fire_alerts(flagged_df, threshold, push_enabled, push_topic, email_enabled, smtp_email, smtp_password, to_email, alert_once):
    """Check flagged stocks and fire alerts for new triggers."""
    new_alerts = []

    for _, row in flagged_df.iterrows():
        ticker = row["Ticker"]

        # Skip if already alerted this session (prevent spam)
        if alert_once and ticker in st.session_state.alerted_tickers:
            continue

        alert_sent = False

        # Push notification
        if push_enabled and push_topic:
            ok = send_push_notification(push_topic, ticker, row["% of ADV"], row["Change %"], row["Price"], threshold)
            if ok:
                alert_sent = True

        # Email
        if email_enabled and smtp_email and smtp_password and to_email:
            ok = send_email_alert(smtp_email, smtp_password, to_email, ticker, row["% of ADV"], row["Change %"], row["Price"], threshold)
            if ok:
                alert_sent = True

        if alert_sent:
            st.session_state.alerted_tickers.add(ticker)
            new_alerts.append({
                "time":    datetime.now().strftime("%H:%M:%S"),
                "ticker":  ticker,
                "adv_pct": row["% of ADV"],
                "change":  row["Change %"],
                "price":   row["Price"],
            })

    st.session_state.alert_log = new_alerts + st.session_state.alert_log
    return new_alerts


# ─── Data Fetch ────────────────────────────────────────────────────────────────
@st.cache_data(ttl=60)
def fetch_data(tickers: list[str]) -> pd.DataFrame:
    rows = []
    for t in tickers:
        try:
            tkr  = yf.Ticker(t)
            info = tkr.info
            fast = tkr.fast_info

            price      = fast.last_price or info.get("regularMarketPrice", 0) or 0
            prev_close = fast.previous_close or info.get("previousClose", 0) or 0
            pm_price   = info.get("preMarketPrice") or price
            pm_vol     = info.get("preMarketVolume") or 0
            avg_vol    = info.get("averageVolume") or info.get("averageDailyVolume10Day") or 1
            float_sh   = info.get("floatShares") or info.get("sharesOutstanding") or 0
            market_cap = info.get("marketCap") or 0
            sector     = info.get("sector") or "Unknown"
            name       = info.get("shortName") or t

            pct_change = ((pm_price - prev_close) / prev_close * 100) if prev_close else 0
            adv_pct    = (pm_vol / avg_vol * 100) if avg_vol else 0

            if   market_cap >= 10e9:  cap = "Large"
            elif market_cap >= 2e9:   cap = "Mid"
            elif market_cap >= 300e6: cap = "Small"
            else:                     cap = "Micro"

            rows.append({
                "Ticker":    t,
                "Name":      name[:22],
                "Price":     round(price, 2),
                "PM Price":  round(pm_price, 2),
                "Change %":  round(pct_change, 2),
                "PM Volume": pm_vol,
                "Avg Vol":   avg_vol,
                "% of ADV":  round(adv_pct, 1),
                "Float":     float_sh,
                "Market Cap":market_cap,
                "Cap Size":  cap,
                "Sector":    sector,
            })
        except Exception:
            continue
    return pd.DataFrame(rows)


def fmt_vol(v):
    if v >= 1e9: return f"{v/1e9:.2f}B"
    if v >= 1e6: return f"{v/1e6:.2f}M"
    if v >= 1e3: return f"{v/1e3:.0f}K"
    return str(int(v))

def flag_icon(pct):
    if pct >= 80: return "🔴"
    if pct >= 50: return "🟠"
    if pct >= 30: return "🟡"
    return "⚪"


# ─── Sidebar ───────────────────────────────────────────────────────────────────
with st.sidebar:
    st.markdown("## ⚙ Scanner Config")
    st.markdown("---")

    # Watchlist
    st.markdown("### 📋 Watchlist")
    ticker_input = st.text_area("Tickers (comma separated)", value=", ".join(DEFAULT_TICKERS), height=100)
    tickers = [t.strip().upper() for t in ticker_input.split(",") if t.strip()]

    st.markdown("---")
    st.markdown("### 🎯 Filter Parameters")

    adv_pct_min = st.slider("Min % of Avg Daily Volume", 0, 150, 30, 5)

    use_change = st.checkbox("Filter by % Change", value=True)
    change_min, change_max = 5.0, 100.0
    if use_change:
        change_min, change_max = st.slider("% Change Range", 0.0, 100.0, (5.0, 100.0), 0.5)

    use_price = st.checkbox("Filter by Price Range", value=False)
    price_min, price_max = 0.0, 10000.0
    if use_price:
        price_min = st.number_input("Min Price ($)", value=1.0, step=0.5)
        price_max = st.number_input("Max Price ($)", value=500.0, step=1.0)

    use_pm_vol = st.checkbox("Filter by Min PM Volume", value=False)
    pm_vol_min = 0
    if use_pm_vol:
        pm_vol_min = st.number_input("Min Pre-Market Volume", value=500000, step=100000)

    use_float = st.checkbox("Filter by Max Float", value=False)
    float_max = 1e12
    if use_float:
        float_max = st.number_input("Max Float (millions)", value=500, step=50) * 1e6

    cap_options  = ["Micro","Small","Mid","Large"]
    selected_caps = st.multiselect("Market Cap", cap_options, default=cap_options)

    all_sectors   = ["Technology","Consumer Cyclical","Healthcare","Financial Services",
                     "Communication Services","Energy","Industrials","Basic Materials","Unknown"]
    selected_sectors = st.multiselect("Sectors (blank = all)", all_sectors, default=[])

    st.markdown("---")

    # ── Alert Settings ─────────────────────────────────────────────────────────
    st.markdown("### 🔔 Alerts")

    alerts_enabled = st.checkbox("Enable Alerts", value=False)

    push_enabled = False
    push_topic   = ""
    email_enabled = False
    smtp_email = smtp_password = to_email = ""

    if alerts_enabled:
        alert_once = st.checkbox("Alert once per ticker per session", value=True,
                                 help="Prevents repeated alerts for the same stock")

        st.markdown("#### 📱 Push Notifications (ntfy.sh)")
        st.caption("Free — install ntfy app on your phone, subscribe to your topic")
        push_enabled = st.checkbox("Enable push notifications", value=False)
        if push_enabled:
            push_topic = st.text_input(
                "Your ntfy topic name",
                placeholder="e.g. damian-scanner-alerts",
                help="Pick any unique name. Subscribe to it in the ntfy app."
            )
            if push_topic:
                st.markdown(
                    f"<div class='alert-box'>📱 Subscribe in the ntfy app to:<br>"
                    f"<b style='color:#3ddf8a'>ntfy.sh/{push_topic}</b></div>",
                    unsafe_allow_html=True
                )

        st.markdown("#### 📧 Email Alerts (Gmail)")
        st.caption("Requires a Gmail App Password — not your regular password")
        email_enabled = st.checkbox("Enable email alerts", value=False)
        if email_enabled:
            smtp_email    = st.text_input("Your Gmail address", placeholder="you@gmail.com")
            smtp_password = st.text_input("Gmail App Password", type="password",
                                          help="Generate at myaccount.google.com → Security → App Passwords")
            to_email      = st.text_input("Send alerts to", placeholder="you@gmail.com")
            if smtp_email and not smtp_password:
                st.markdown(
                    "<div class='alert-box-warn'>⚠ You need a Gmail <b>App Password</b>, not your regular password.<br>"
                    "Go to: myaccount.google.com → Security → 2-Step Verification → App Passwords</div>",
                    unsafe_allow_html=True
                )
    else:
        alert_once = True

    st.markdown("---")
    st.markdown("### 🔄 Auto-Refresh")
    auto_refresh    = st.checkbox("Auto-refresh every 60s", value=False)
    refresh_seconds = 60
    if auto_refresh:
        refresh_seconds = st.slider("Refresh interval (seconds)", 30, 300, 60, 30)

    st.markdown("---")
    if st.button("🗑 Reset Alert History"):
        st.session_state.alerted_tickers = set()
        st.session_state.alert_log = []
        st.success("Alert history cleared")

    st.caption("Data via yfinance · Pre-market 4:00–9:30 AM ET")


# ─── Header ───────────────────────────────────────────────────────────────────
col_title, col_time = st.columns([3, 1])
with col_title:
    st.markdown('<h1><span class="live-dot"></span>Pre-Market Scanner</h1>', unsafe_allow_html=True)
with col_time:
    st.markdown(
        f"<div style='text-align:right;color:#2e3a55;font-size:11px;margin-top:22px;'>"
        f"{datetime.now().strftime('%H:%M:%S')}</div>",
        unsafe_allow_html=True
    )

col_btn, col_note = st.columns([1, 3])
with col_btn:
    scan_clicked = st.button("⟳  RUN SCAN", use_container_width=True)
with col_note:
    alert_status = ""
    if alerts_enabled:
        channels = []
        if push_enabled and push_topic: channels.append("📱 Push")
        if email_enabled and to_email:  channels.append("📧 Email")
        if channels:
            alert_status = f" · Alerts: <span style='color:#3ddf8a'>{' + '.join(channels)}</span>"
        else:
            alert_status = " · <span style='color:#f5a623'>⚠ Alerts on but not configured</span>"

    st.markdown(
        f"<div style='color:#3a4a65;font-size:11px;margin-top:10px;'>"
        f"Scanning {len(tickers)} tickers · Core signal ≥ <b style='color:#f5c842'>{adv_pct_min}%</b> of ADV"
        f"{alert_status}</div>",
        unsafe_allow_html=True
    )

st.markdown("---")

# ─── Main Scan Logic ──────────────────────────────────────────────────────────
if scan_clicked or auto_refresh:
    with st.spinner("📡 Fetching pre-market data..."):
        df = fetch_data(tickers)

    if df.empty:
        st.error("No data returned. Check your tickers or internet connection.")
        st.stop()

    # Apply filters
    mask = df["% of ADV"] >= adv_pct_min
    if use_change:    mask &= (df["Change %"] >= change_min) & (df["Change %"] <= change_max)
    if use_price:     mask &= (df["Price"] >= price_min) & (df["Price"] <= price_max)
    if use_pm_vol:    mask &= df["PM Volume"] >= pm_vol_min
    if use_float:     mask &= df["Float"] <= float_max
    if selected_caps: mask &= df["Cap Size"].isin(selected_caps)
    if selected_sectors: mask &= df["Sector"].isin(selected_sectors)

    flagged    = df[mask].sort_values("% of ADV", ascending=False)
    all_sorted = df.sort_values("% of ADV", ascending=False)

    # ── Fire Alerts ────────────────────────────────────────────────────────────
    if alerts_enabled and not flagged.empty:
        new_alerts = fire_alerts(
            flagged, adv_pct_min,
            push_enabled, push_topic,
            email_enabled, smtp_email, smtp_password, to_email,
            alert_once
        )
        if new_alerts:
            st.success(f"🔔 {len(new_alerts)} alert(s) sent for: {', '.join(a['ticker'] for a in new_alerts)}")

    # ── Metrics ────────────────────────────────────────────────────────────────
    m1, m2, m3, m4 = st.columns(4)
    m1.metric("Tickers Scanned", len(df))
    m2.metric("Flagged",         len(flagged))
    m3.metric("Avg ADV %",       f"{df['% of ADV'].mean():.1f}%")
    m4.metric("Peak ADV %",      f"{df['% of ADV'].max():.1f}%")

    st.markdown("---")

    # ── Alert Log ──────────────────────────────────────────────────────────────
    if st.session_state.alert_log:
        with st.expander(f"🔔 Alert Log  ({len(st.session_state.alert_log)} sent this session)", expanded=False):
            log_df = pd.DataFrame(st.session_state.alert_log)
            log_df.columns = ["Time","Ticker","% of ADV","Change %","Price"]
            st.dataframe(log_df, use_container_width=True, hide_index=True)

    # ── Flagged Table ──────────────────────────────────────────────────────────
    st.markdown(
        f"### 🔴 Flagged Stocks  "
        f"<span style='color:#3a4a65;font-size:13px;'>({len(flagged)} results)</span>",
        unsafe_allow_html=True
    )

    if flagged.empty:
        st.info("No stocks match your filters. Try lowering the ADV % threshold in the sidebar.")
    else:
        display = flagged.copy()
        display["Signal"]    = display["% of ADV"].apply(flag_icon)
        display["Alerted"]   = display["Ticker"].apply(lambda t: "✅" if t in st.session_state.alerted_tickers else "")
        display["PM Volume"] = display["PM Volume"].apply(fmt_vol)
        display["Avg Vol"]   = display["Avg Vol"].apply(fmt_vol)
        display["Float"]     = display["Float"].apply(fmt_vol)
        display["Change %"]  = display["Change %"].apply(lambda x: f"+{x:.2f}%" if x>=0 else f"{x:.2f}%")
        display["Price"]     = display["Price"].apply(lambda x: f"${x:.2f}")
        display["PM Price"]  = display["PM Price"].apply(lambda x: f"${x:.2f}")

        cols = ["Signal","Ticker","Name","Price","PM Price","Change %","PM Volume","Avg Vol","% of ADV","Float","Cap Size","Sector","Alerted"]
        st.dataframe(
            display[cols].reset_index(drop=True),
            use_container_width=True,
            height=min(60 + len(flagged) * 38, 520),
            column_config={
                "Signal":   st.column_config.TextColumn("",        width=40),
                "Alerted":  st.column_config.TextColumn("Alerted", width=65),
                "% of ADV": st.column_config.ProgressColumn("% of ADV", min_value=0, max_value=150, format="%.1f%%"),
            }
        )
        csv = flagged.to_csv(index=False)
        st.download_button("⬇ Export to CSV", data=csv,
            file_name=f"pm_scan_{datetime.now().strftime('%Y%m%d_%H%M')}.csv", mime="text/csv")

    st.markdown("---")

    # ── Full Watchlist ─────────────────────────────────────────────────────────
    with st.expander("📋 Full Watchlist", expanded=False):
        full = all_sorted.copy()
        full["Signal"]   = full["% of ADV"].apply(flag_icon)
        full["PM Volume"]= full["PM Volume"].apply(fmt_vol)
        full["Avg Vol"]  = full["Avg Vol"].apply(fmt_vol)
        full["Change %"] = full["Change %"].apply(lambda x: f"+{x:.2f}%" if x>=0 else f"{x:.2f}%")
        full["Price"]    = full["Price"].apply(lambda x: f"${x:.2f}")
        cols_f = ["Signal","Ticker","Name","Price","Change %","PM Volume","Avg Vol","% of ADV","Cap Size","Sector"]
        st.dataframe(full[cols_f].reset_index(drop=True), use_container_width=True,
            column_config={
                "Signal":   st.column_config.TextColumn("", width=40),
                "% of ADV": st.column_config.ProgressColumn("% of ADV", min_value=0, max_value=150, format="%.1f%%"),
            })

    # ── Chart ──────────────────────────────────────────────────────────────────
    with st.expander("📊 ADV % Chart", expanded=True):
        st.bar_chart(all_sorted.set_index("Ticker")[["% of ADV"]].head(20), color="#1a6aff")

    # ── Auto-refresh ──────────────────────────────────────────────────────────
    if auto_refresh:
        st.info(f"⏱ Auto-refreshing every {refresh_seconds}s...")
        time.sleep(refresh_seconds)
        st.cache_data.clear()
        st.rerun()

else:
    # Landing
    st.markdown("""
    <div style='text-align:center;padding:60px 20px;color:#2a3858;'>
        <div style='font-size:48px;margin-bottom:16px;'>📡</div>
        <div style='font-size:18px;font-weight:700;color:#3a4a65;'>Scanner Ready</div>
        <div style='font-size:12px;margin-top:8px;'>Configure filters + alerts in the sidebar, then hit <b style="color:#7ab0f0">RUN SCAN</b></div>
        <br>
        <div style='font-size:11px;color:#1e2840;'>
            📱 Push alerts via ntfy.sh · Free · No account needed<br>
            📧 Email alerts via Gmail · Requires App Password
        </div>
    </div>
    """, unsafe_allow_html=True)
