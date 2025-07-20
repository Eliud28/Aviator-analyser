import time
import csv
import os

# === Configuration ===
STARTING_BALANCE = 1000
BASE_BET = 10
AUTO_CASHOUT = 2.0
MAX_ROUNDS = 50
SKIP_AFTER_HIGH = True
HIGH_THRESHOLD = 10.0
LOG_FILE = "aviator_log.csv"

# === Bot State ===
balance = STARTING_BALANCE
bet = BASE_BET
round_history = []
skip_next = False

# === Prepare CSV Logging ===
write_header = not os.path.exists(LOG_FILE)
with open(LOG_FILE, mode='a', newline='') as file:
    writer = csv.writer(file)
    if write_header:
        writer.writerow(["Round", "Multiplier", "Bet", "Result", "Balance"])

print("Aviator Prediction Bot Starting (Real-Time Input Mode)...")
print(f"Starting Balance: {balance} KES\n")

for round_number in range(1, MAX_ROUNDS + 1):
    print(f"--- Round {round_number} ---")

    try:
        crash_multiplier = float(input("Enter actual crash multiplier (e.g. 1.67): "))
    except ValueError:
        print("Invalid input. Skipping round.\n")
        continue

    round_history.append(crash_multiplier)

    if skip_next:
        print("Skipping round due to high previous multiplier.\n")
        skip_next = False
        continue

    if balance < bet:
        print("Insufficient balance. Ending simulation.")
        break

    balance -= bet
    print(f"Bet: {bet} KES at auto cashout {AUTO_CASHOUT}x")

    if crash_multiplier >= AUTO_CASHOUT:
        win_amount = bet * AUTO_CASHOUT
        balance += win_amount
        result = "Win"
        print(f"✅ Won: {win_amount} KES | New Balance: {balance} KES")
        bet = BASE_BET
    else:
        result = "Loss"
        print(f"❌ Lost: {bet} KES | New Balance: {balance} KES")
        bet *= 2

    if SKIP_AFTER_HIGH and crash_multiplier >= HIGH_THRESHOLD:
        skip_next = True

    # Log to CSV
    with open(LOG_FILE, mode='a', newline='') as file:
        writer = csv.writer(file)
        writer.writerow([round_number, crash_multiplier, bet, result, balance])

    print()
    time.sleep(1)

print("--- Simulation Ended ---")
print(f"Final Balance: {balance} KES")
print(f"Round history logged in: {LOG_FILE}")
import csv
import matplotlib.pyplot as plt
import pandas as pd
import seaborn as sns
import numpy as np
import streamlit as st
import os

LOG_FILE = "aviator_log.csv"
EXPORT_FILE = "aviator_summary.xlsx"


def load_data(log_file):
    rounds = []
    multis = []
    balances = []
    results = []
    profits = []
    with open(log_file, newline='') as csvfile:
        reader = csv.DictReader(csvfile)
        previous_balance = None
        for row in reader:
            rounds.append(int(row['Round']))
            multis.append(float(row['Multiplier']))
            balances.append(float(row['Balance']))
            results.append(row['Result'])
            if previous_balance is not None:
                profits.append(float(row['Balance']) - previous_balance)
            else:
                profits.append(0)
            previous_balance = float(row['Balance'])
    return rounds, multis, balances, results, profits


def analyze_csv(rounds, multis, balances, results, profits):
    wins = results.count('Win')
    losses = results.count('Loss')
    win_streak = 0
    max_win_streak = 0
    loss_streak = 0
    max_loss_streak = 0
    consecutive_high = 0
    pattern_alerts = []
    multi_streaks = []
    HIGH_MULTI_THRESHOLD = 5.0

    for i, result in enumerate(results):
        if result == 'Win':
            win_streak += 1
            loss_streak = 0
            max_win_streak = max(max_win_streak, win_streak)
        else:
            loss_streak += 1
            win_streak = 0
            max_loss_streak = max(max_loss_streak, loss_streak)

        if i >= 3 and results[i-3:i] == ['Loss', 'Loss', 'Loss'] and result == 'Win':
            pattern_alerts.append(f"Round {rounds[i]}: Win after 3 losses")

        if multis[i] >= HIGH_MULTI_THRESHOLD:
            consecutive_high += 1
            if consecutive_high == 3:
                multi_streaks.append(f"Round {rounds[i]}: 3+ multipliers >= {HIGH_MULTI_THRESHOLD}x")
        else:
            consecutive_high = 0

    print("\n=== Aviator Log Summary ===")
    print(f"Total Rounds: {len(rounds)}")
    print(f"Wins: {wins} | Losses: {losses} | Win Rate: {wins / len(rounds) * 100:.2f}%")
    print(f"Max Win Streak: {max_win_streak} | Max Loss Streak: {max_loss_streak}")
    print(f"Final Balance: {balances[-1]} | Average Multiplier: {sum(multis) / len(multis):.2f}")
    print(f"Total Profit: {balances[-1] - balances[0]:.2f}")

    if pattern_alerts or multi_streaks:
        print("\n🔍 Detected Patterns:")
        for alert in pattern_alerts + multi_streaks:
            print(f"- {alert}")

    if loss_streak >= 2:
        print("\n📊 Suggestion: Loss streak — consider low stakes or skipping.")
    elif win_streak >= 3:
        print("\n📊 Suggestion: Win streak — trend continuation possible.")
    elif consecutive_high >= 2:
        print("\n📊 Suggestion: High volatility — consider risk management.")
    else:
        print("\n📊 Suggestion: No clear trend — use default strategy.")

    df = pd.DataFrame({
        'Round': rounds,
        'Multiplier': multis,
        'Result': results,
        'Balance': balances,
        'Profit': profits
    })
    df.to_excel(EXPORT_FILE, index=False)
    print(f"\n✅ Exported to: {EXPORT_FILE}")

    plt.figure(figsize=(10, 5))
    plt.plot(rounds, balances, label='Balance', color='green')
    plt.bar(rounds, profits, alpha=0.4, label='Profit', color='blue')
    plt.title('Aviator Game Performance')
    plt.xlabel('Round')
    plt.ylabel('KES')
    plt.legend()
    plt.grid(True)
    plt.tight_layout()
    plt.show()

    plt.figure(figsize=(10, 3))
    bins = [0, 2, 5, 10, 20, 50, 100, 1000]
    labels = ['<2x', '2-5x', '5-10x', '10-20x', '20-50x', '50-100x', '>100x']
    grouped = pd.cut(multis, bins=bins, labels=labels, include_lowest=True)
    counts = pd.value_counts(grouped, sort=False)
    sns.heatmap([counts.values], annot=True, fmt='d', cmap='YlGnBu', xticklabels=labels)
    plt.title('Multiplier Frequency Heatmap')
    plt.yticks([])
    plt.tight_layout()
    plt.show()


def simulate_cashout(multis, stake=10, levels=[1.5, 2.0, 3.0, 5.0]):
    print("\n📈 Cashout Strategy Simulation:")
    for level in levels:
        profit = sum(stake * level if m >= level else -stake for m in multis)
        print(f"  Cashout at {level}x → Net Profit: {profit:.2f} KES")


def simulate_martingale(multis, base_bet=10, target=2.0, max_rounds=10):
    print("\n🎲 Martingale Strategy Simulation:")
    balance = 0
    bet = base_bet
    for m in multis:
        if m >= target:
            balance += bet * target
            bet = base_bet
        else:
            balance -= bet
            bet *= 2
            if bet > base_bet * (2 ** max_rounds):
                print("  💥 Bankroll limit exceeded — reset")
                bet = base_bet
    print(f"Final balance: {balance:.2f} KES")


def launch_web_ui():
    st.title("🕹️ Aviator Game Analyzer")
    uploaded_file = st.file_uploader("Upload aviator_log.csv", type=["csv"])
    if uploaded_file:
        df = pd.read_csv(uploaded_file)
        rounds = df['Round'].tolist()
        multis = df['Multiplier'].tolist()
        results = df['Result'].tolist()
        balances = df['Balance'].tolist()
        profits = [balances[i] - balances[i - 1] if i > 0 else 0 for i in range(len(balances))]
        st.write("### Game Summary")
        st.write(df.describe())
        st.line_chart(df[['Balance']])
        st.bar_chart(df[['Profit']])
        st.download_button("Download Summary", df.to_csv(index=False), "summary.csv")


def main():
    print("""
  === Aviator Strategy Toolkit ===
  1. Analyze CSV and show stats
  2. Simulate custom cashout strategy
  3. Simulate Martingale strategy
  4. Launch Web Dashboard
  5. Exit
    """)
    choice = input("Select option: ")
    if choice == '1':
        data = load_data(LOG_FILE)
        analyze_csv(*data)
    elif choice == '2':
        levels = input("Enter cashout levels (comma-separated, e.g. 1.5,2.5,5): ")
        levels = [float(x.strip()) for x in levels.split(",")]
        stake = float(input("Enter stake amount (KES): "))
        _, multis, _, _, _ = load_data(LOG_FILE)
        simulate_cashout(multis, stake, levels)
    elif choice == '3':
        base = float(input("Base bet (KES): "))
        target = float(input("Target multiplier (e.g. 2.0): "))
        max_r = int(input("Max martingale rounds: "))
        _, multis, _, _, _ = load_data(LOG_FILE)
        simulate_martingale(multis, base, target, max_r)
    elif choice == '4':
        launch_web_ui()
    else:
        print("Goodbye!")


if __name__ == "__main__":
    main(
    
