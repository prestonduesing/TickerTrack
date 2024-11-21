from flask import Flask, render_template, request, redirect, url_for
import yfinance as yf
import plotly.graph_objs as go
import plotly.io as pio
import json

app = Flask(__name__)

# Function to format large numbers with suffixes
def format_market_cap(value):
    if value >= 1_000_000_000_000:
        return f"{value / 1_000_000_000_000:.2f} T"
    elif value >= 1_000_000_000:
        return f"{value / 1_000_000_000:.2f} B"
    elif value >= 1_000_000:
        return f"{value / 1_000_000:.2f} M"
    else:
        return f"{value:,}"

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/results', methods=['POST'])
def results():
    ticker = request.form.get('ticker')
    if not ticker:
        return redirect(url_for('index'))
    
    stock = yf.Ticker(ticker)
    try:
        data = stock.history(period="1y")
        info = stock.info
    except Exception:
        return render_template('index.html', error="Invalid ticker symbol or data not available")
    
    stock_name = info.get("shortName", "N/A")
    
    # Retrieve and format market cap
    market_cap = info.get("marketCap", "N/A")
    if isinstance(market_cap, (int, float)):
        market_cap = format_market_cap(market_cap)
    
    industry = info.get("industry", "N/A")
    pe_ratio = info.get("trailingPE", "N/A")
    
    # Price Line Chart with Moving Averages
    price_fig = go.Figure()
    price_fig.add_trace(go.Scatter(x=data.index, y=data['Close'], mode='lines', name="Close Price", line=dict(color="blue", width=2)))
    price_fig.add_trace(go.Scatter(x=data.index, y=data['Close'].rolling(window=20).mean(), mode='lines', name="20-Day SMA", line=dict(color="orange", width=2, dash="dash")))
    price_fig.add_trace(go.Scatter(x=data.index, y=data['Close'].ewm(span=20, adjust=False).mean(), mode='lines', name="20-Day EMA", line=dict(color="green", width=2, dash="dot")))
    price_fig.update_layout(
        title=f'{stock_name} Stock Price with Moving Averages',
        xaxis_title='Date',
        yaxis_title='Price',
        plot_bgcolor="#f4f4f9",
        paper_bgcolor="#f4f4f9",
        font=dict(family="Arial", size=14),
        legend=dict(
            orientation="h",
            yanchor="bottom",
            y=1.02,
            xanchor="right",
            x=1
        ),
        autosize=True,
        margin=dict(l=20, r=20, t=50, b=50)
    )
    price_graphJSON = pio.to_json(price_fig)

    # Volume Bar Chart
    volume_fig = go.Figure()
    volume_fig.add_trace(go.Bar(x=data.index, y=data['Volume'], name="Volume", marker=dict(color="purple")))
    volume_fig.update_layout(
        title=f'{stock_name} Trading Volume',
        xaxis_title='Date',
        yaxis_title='Volume',
        plot_bgcolor="#f4f4f9",
        paper_bgcolor="#f4f4f9",
        font=dict(family="Arial", size=14),
        legend=dict(
            orientation="h",
            yanchor="bottom",
            y=1.02,
            xanchor="right",
            x=1
        ),
        autosize=True,
        margin=dict(l=20, r=20, t=50, b=50)
    )
    volume_graphJSON = pio.to_json(volume_fig)

    # High-Low Range Chart
    high_low_fig = go.Figure()
    high_low_fig.add_trace(go.Scatter(x=data.index, y=data['High'], mode='lines', name="High Price", line=dict(color="green", width=1)))
    high_low_fig.add_trace(go.Scatter(x=data.index, y=data['Low'], mode='lines', name="Low Price", line=dict(color="red", width=1)))
    high_low_fig.add_trace(go.Scatter(x=data.index, y=data['Close'], mode='lines', name="Close Price", line=dict(color="blue", width=2, dash="dot")))
    high_low_fig.update_layout(
        title=f'{stock_name} Daily High-Low Range',
        xaxis_title='Date',
        yaxis_title='Price',
        plot_bgcolor="#f4f4f9",
        paper_bgcolor="#f4f4f9",
        font=dict(family="Arial", size=14),
        legend=dict(
            orientation="h",
            yanchor="bottom",
            y=1.02,
            xanchor="right",
            x=1
        ),
        autosize=True,
        margin=dict(l=20, r=20, t=50, b=50)
    )
    high_low_graphJSON = pio.to_json(high_low_fig)

    # Candlestick Chart
    candlestick_fig = go.Figure()
    candlestick_fig.add_trace(go.Candlestick(
        x=data.index,
        open=data['Open'],
        high=data['High'],
        low=data['Low'],
        close=data['Close'],
        name="Candlestick"
    ))
    candlestick_fig.update_layout(
        title=f'{stock_name} Daily Candlestick Chart',
        xaxis_title='Date',
        yaxis_title='Price',
        plot_bgcolor="#f4f4f9",  # Match background color with other charts
        paper_bgcolor="#f4f4f9",
        font=dict(family="Arial", size=14),
        legend=dict(
            orientation="h",
            yanchor="bottom",
            y=1.02,
            xanchor="right",
            x=1
        ),
        autosize=True,
        margin=dict(l=20, r=20, t=50, b=50),
        xaxis=dict(
            rangeslider=dict(visible=False),  # Hide the rangeslider at the bottom
            showgrid=True
        )
    )
    candlestick_graphJSON = pio.to_json(candlestick_fig)

    return render_template(
        'results.html',
        ticker=ticker,
        stock_name=stock_name,
        market_cap=market_cap,
        industry=industry,
        pe_ratio=pe_ratio,
        price_graphJSON=price_graphJSON,
        volume_graphJSON=volume_graphJSON,
        high_low_graphJSON=high_low_graphJSON,
        candlestick_graphJSON=candlestick_graphJSON
    )

if __name__ == '__main__':
    app.run(debug=True)
