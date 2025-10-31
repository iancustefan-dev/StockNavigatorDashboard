import streamlit as st
import pandas as pd
import numpy as np
import plotly.express as px
import plotly.graph_objects as go

# ----------------------------------------------
# 📥 Load Data
# ----------------------------------------------
def load_portfolio_data():
    try:
        df = pd.read_json('scoruri_live.json')
    except Exception:
        df = pd.read_csv('portfolio_scores.csv')
    return df

# ----------------------------------------------
# ⚙️ Core Functions
# ----------------------------------------------
def calculate_alerts(df):
    alerts = []
    for _, row in df.iterrows():
        if row['Score'] < 5.0:
            alerts.append(f"⚠️ {row['Symbol']}: Score < 5.0 → SELL signal")
        elif abs(row['Score_Change']) > 0.8:
            alerts.append(f"🔔 {row['Symbol']}: Δ Score {row['Score_Change']:+.2f} → Review position")
    return alerts

def plot_sector_allocation(df):
    sector_summary = df.groupby('Sector')['Weight'].sum().reset_index()
    fig = px.pie(sector_summary, values='Weight', names='Sector', title='Sector Allocation', hole=0.4)
    return fig

def plot_score_heatmap(df):
    pivot = df.pivot(index='Symbol', columns='Category', values='Score_Component')
    fig = px.imshow(pivot, color_continuous_scale='RdYlGn', title='Score Breakdown Heatmap')
    return fig

def plot_vix_history(vix_df):
    fig = go.Figure()
    fig.add_trace(go.Scatter(x=vix_df['Date'], y=vix_df['VIX'], mode='lines', name='VIX'))
    fig.add_hline(y=25, line_dash='dash', line_color='red', annotation_text='High Volatility Threshold')
    fig.update_layout(title='VIX History & Circuit Breaker', yaxis_title='VIX Level')
    return fig

# ----------------------------------------------
# 🖥️ Streamlit App
# ----------------------------------------------
st.set_page_config(page_title="Portfolio Scoring System v2.2", layout="wide")
st.title("📊 Portfolio Scoring System v2.2 – Live Dashboard")

# Load data
df = load_portfolio_data()
df['Score_Change'] = df['Score'] - df['Prev_Score']

# Sidebar
st.sidebar.header("⚙️ Controls")
view = st.sidebar.selectbox("View", ["Overview", "Scores", "Alerts", "Risk Monitor"])
vix_level = st.sidebar.number_input("Current VIX", min_value=10.0, max_value=80.0, value=18.0)

# Overview Page
if view == "Overview":
    st.subheader("Portfolio Overview")
    col1, col2 = st.columns(2)
    with col1:
        st.metric("Average Score", f"{df['Score'].mean():.2f}")
        st.metric("Sharpe Estimate", f"{(df['Score'].mean()/df['Volatility'].mean()):.2f}")
    with col2:
        fig_sector = plot_sector_allocation(df)
        st.plotly_chart(fig_sector, use_container_width=True)

    st.dataframe(df[['Symbol', 'Company', 'Sector', 'Score', 'Verdict']].sort_values('Score', ascending=False).head(10))

# Scores Page
elif view == "Scores":
    st.subheader("Detailed Scores")
    fig_heatmap = plot_score_heatmap(df)
    st.plotly_chart(fig_heatmap, use_container_width=True)
    st.dataframe(df[['Symbol', 'Fundamental', 'Technical', 'Macro', 'Sentiment', 'Risk', 'Score']])

# Alerts Page
elif view == "Alerts":
    st.subheader("📢 Portfolio Alerts")
    alerts = calculate_alerts(df)
    if alerts:
        for alert in alerts:
            st.warning(alert)
    else:
        st.success("No alerts – portfolio stable.")

# Risk Monitor
elif view == "Risk Monitor":
    st.subheader("📈 Risk & VIX Monitor")
    try:
        vix_df = pd.read_csv('vix_history.csv')
        fig_vix = plot_vix_history(vix_df)
        st.plotly_chart(fig_vix, use_container_width=True)
    except Exception:
        st.info("Upload 'vix_history.csv' for VIX visualization.")

    if vix_level > 25:
        st.error("🚨 Circuit Breaker Active – Freeze Rebalancing")
    else:
        st.success("✅ Normal trading regime")
