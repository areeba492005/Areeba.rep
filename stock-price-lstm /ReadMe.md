import numpy as np
import matplotlib.pyplot as plt
import yfinance as yf
from sklearn.preprocessing import MinMaxScaler
from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import LSTM, Dense, Dropout

df = yf.download('AAPL', start='2018-01-01', end='2024-01-01')

data = df['Close'].values.reshape(-1, 1)
scaler = MinMaxScaler()
data = scaler.fit_transform(data)

def make_sequences(data, steps=60):
    X, y = [], []
    for i in range(steps, len(data)):
        X.append(data[i-steps:i])
        y.append(data[i])
    return np.array(X), np.array(y)

X, y = make_sequences(data)
split = int(len(X) * 0.8)
X_train, X_test = X[:split], X[split:]
y_train, y_test = y[:split], y[split:]

model = Sequential([
LSTM(64, return_sequences=True, input_shape=(60, 1)),
Dropout(0.2),
LSTM(32),
Dropout(0.2),
Dense(1)
])
model.compile(optimizer='adam', loss='mse')
model.fit(X_train, y_train, epochs=20, batch_size=32, validation_split=0.1)

pred = scaler.inverse_transform(model.predict(X_test))
actual = scaler.inverse_transform(y_test)

plt.figure(figsize=(12,5))
plt.plot(actual, label='Actual')
plt.plot(pred, label='Predicted')
plt.title('Actual vs Predicted')
plt.legend()
plt.show()
