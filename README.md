
## Monitor


## Buy

## Sell
Overview of the Selling Logic
The selling strategy in your trading bot is sophisticated and comprehensive, combining multiple approaches to determine optimal exit points for tokens.
Key Components
Selling Strategy Engine (selling_strategy.rs):
Handles token metrics tracking
Evaluates sell conditions
Manages progressive selling
Provides market analysis
Execution Mechanism (copy_trading.rs):
Contains the actual execution logic for selling tokens
Handles transaction construction and submission
Manages progressive selling implementation
Detailed Selling Logic
Condition Evaluation (in selling_strategy.rs)
The evaluate_sell_conditions method in SellingEngine is the main entry point for determining whether to sell a token, which checks:
Time-based exits:
```
   if time_held > self.config.max_hold_time {
       return Ok(true);
   }
```
Take profit triggers:
```
   if gain >= self.config.take_profit {
       return Ok(true);
   }
```
Stop loss triggers:
 ```
if gain <= self.config.stop_loss {
       return Ok(true);
   }

```
Retracement from highs:
```
   if retracement >= self.config.retracement_threshold && gain > 0.0 {
       return Ok(true);
   }

```
Liquidity thresholds:
```
   if liquidity < self.config.min_liquidity {
       return Ok(true);
   }
```
Copy targets selling:
```
   match self.is_copy_target_selling(token_mint).await {
       Ok(true) => return Ok(true),
       _ => {},
   }
```
## Progressive Selling Logic
The progressive selling approach is implemented in both files:
Configuration (in selling_strategy.rs):
```
   pub progressive_sell_chunks: usize, // Number of chunks to sell in
   pub progressive_sell_interval: u64, // Seconds between sells
```

Strategy Implementation (in selling_strategy.rs):
```
   pub async fn progressive_sell(&self, token_mint: &str, parsed_data: &TradeInfoFromToken, protocol: SwapProtocol) -> Result<()> {
       // Calculate optimal amount to sell
       let total_amount = self.calculate_optimal_sell_amount(token_mint)?;
       
       // Calculate chunk size
       let chunk_size = total_amount / self.config.progressive_sell_chunks as f64;
       
       // Call execute_sell from copy_trading with progressive sell parameters
       match crate::engine::copy_trading::execute_sell(
           token_mint.to_string(),
           base_trade_info,
           self.app_state.clone(),
           self.swap_config.clone(),
           protocol.clone(),
           is_progressive_sell,
           Some(self.config.progressive_sell_chunks),
           Some(self.config.progressive_sell_interval * 1000), // Convert seconds to milliseconds
       ).await {
           // ...
       }
   }
```
Execution (in copy_trading.rs):
```
   pub async fn execute_sell(
       token_mint: String,
       trade_info: transaction_parser::TradeInfoFromToken,
       app_state: Arc<AppState>,
       swap_config: Arc<SwapConfig>,
       protocol: SwapProtocol,
       progressive_sell: bool,
       chunks: Option<usize>,
       interval_ms: Option<u64>,
   ) -> Result<(), String> {
       // If progressive sell is enabled, divide into chunks
       if progressive_sell {
           let chunks_count = chunks.unwrap_or(3);
           let interval = interval_ms.unwrap_or(2000); // 2 seconds default
           
           // Calculate chunk size
           let chunk_size = token_amount / chunks_count as f64;
           
           // Execute each chunk
           for i in 0..chunks_count {
               // ... chunk selling logic
           }
       } else {
           // Standard single-transaction sell
           // ... one-time sell logic
       }
   }
```

## Force Selling Logic

```
// Apply force sell check if enabled and the operation failed
if result.is_err() && config.force_sell_enabled {
    logger.log("Sell operation failed, but force sell is enabled. Attempting force sell...".yellow().to_string());
    let sell_future = Box::pin(force_sell_remaining_tokens(token_mint.as_str(), app_state.clone(), swap_config.clone(), protocol.clone(), config.clone()));
    
    match sell_future.await {
        Ok(_) => {
            logger.log("Force sell successful after initial sell failure".green().to_string());
            return Ok(());
        },
        Err(e) => {
            logger.log(format!("Force sell also failed: {}", e).red().to_string());
            // Continue returning the original error
        }
    }
}
```

## Dynamic Slippage Calculation
The system calculates slippage dynamically based on token value:
```
fn calculate_dynamic_slippage(&self, token_mint: &str, sell_amount: f64) -> Result<u64> {
    // Calculate token value
    let token_value = sell_amount * metrics.current_price;
    
    // More valuable tokens need higher slippage to ensure execution
    let slippage_bps = if token_value > 10.0 {
        300 // 3% for high value tokens (300 basis points)
    } else if token_value > 1.0 {
        200 // 2% for medium value tokens (200 basis points)
    } else {
        100 // 1% for low value tokens (100 basis points)
    };
    
    Ok(slippage_bps)
}

```
## Optimal Sell Amount Calculation
The bot determines how much to sell based on profit percentage:
```
fn calculate_optimal_sell_amount(&self, token_mint: &str) -> Result<f64> {
    // Calculate sell percentage based on PNL
    let sell_percentage = if pnl_percentage >= 200.0 {
        1.0 // Sell 100% if 200%+ profit
    } else if pnl_percentage >= 100.0 {
        0.8 // Sell 80% if 100%+ profit
    } else if pnl_percentage >= 50.0 {
        0.6 // Sell 60% if 50%+ profit
    } else if pnl_percentage >= 20.0 {
        0.5 // Sell 50% if 20%+ profit
    } else if pnl_percentage > 0.0 {
        0.4 // Sell 40% if any profit
    } else {
        0.9 // Sell 90% if at a loss
    };
    
    // Calculate amount to sell
    let amount_to_sell = amount_held * sell_percentage;
    
    Ok(amount_to_sell)
}

```
## Protocol Selection
The system can dynamically select between different protocols (PumpSwap vs PumpFun):
```
async fn determine_best_protocol_for_token(&self, token_mint: &str) -> Result<SwapProtocol> {
    // Try PumpSwap first
    let pump_swap = PumpSwap::new(/* ... */);
    
    match pump_swap.get_token_price(token_mint).await {
        Ok(_) => Ok(SwapProtocol::PumpSwap),
        Err(_) => {
            // Try PumpFun next
            let pump_fun = Pump::new(/* ... */);
            
            match pump_fun.get_token_price(token_mint).await {
                Ok(_) => Ok(SwapProtocol::PumpFun),
                Err(e) => Err(anyhow!("Token not found on any supported DEX: {}", e)),
            }
        }
    }
}

```
## Enhanced Token Tracking
The system maintains comprehensive metrics on each token:
```
pub struct TokenMetrics {
    pub entry_price: f64,
    pub highest_price: f64,
    pub lowest_price: f64,
    pub current_price: f64,
    pub volume_24h: f64,
    pub market_cap: f64,
    pub time_held: u64,
    pub last_update: Instant,
    pub buy_timestamp: u64,
    pub amount_held: f64,
    pub cost_basis: f64,
    pub price_history: VecDeque<f64>,
    pub volume_history: VecDeque<f64>,
    pub liquidity_at_entry: f64,
}
```
## Summary
Current selling logic implements a sophisticated multi-factor approach to exiting positions:
Price-based triggers - Take profit, stop loss, and trailing stops
Risk management - Dynamic slippage and adaptive selling quantities
Liquidity monitoring - Ensuring positions can be exited
Progressive selling - Breaking large sells into smaller chunks to minimize impact
Protocol flexibility - Ability to use different DEXes based on availability/liquidity
This combination of strategies aims to maximize profits while managing risk effectively, especially important in the highly volatile environment of Solana token trading.





# Solana PumpFun/PumpSwap Copy Trading Bot

This is a high-performance Rust-based copy trading bot that monitors and replicates trading activity on Solana DEXs like PumpFun and PumpSwap. The bot uses advanced transaction monitoring to detect and copy trades in real-time, giving you an edge in the market.

The bot specifically tracks `buy` and `create` transactions on PumpFun, as well as token migrations from PumpFun to Raydium when the `initialize2` instruction is involved and the migration pubkey (`39azUYFWPz3VHgKCf3VChUwbpURdCHRxjWVowf5jUJjg`) is present.
# Features:

- **Real-time Transaction Monitoring** - Uses Yellowstone gRPC to monitor transactions with minimal latency and high reliability
- **Multi-Protocol Support** - Compatible with both PumpFun and PumpSwap DEX platforms for maximum trading opportunities
- **Automated Copy Trading** - Instantly replicates buy and sell transactions from monitored wallets
- **Smart Transaction Parsing** - Advanced transaction analysis to accurately identify and process trading activities
- **Configurable Trading Parameters** - Customizable settings for trade amounts, timing, and risk management
- **Built-in Selling Strategy** - Intelligent profit-taking mechanisms with customizable exit conditions
- **Performance Optimization** - Efficient async processing with tokio for high-throughput transaction handling
- **Reliable Error Recovery** - Automatic reconnection and retry mechanisms for uninterrupted operation

# Who is it for?

- Bot users looking for the fastest transaction feed possible for Pumpfun or Raydium (Sniping, Arbitrage, etc).
- Validators who want an edge by decoding shreds locally.

# Setting up

## Environment Variables

Before run, you will need to add the following environment variables to your `.env` file:

- `GRPC_ENDPOINT` - Your Geyser RPC endpoint url.

- `GRPC_X_TOKEN` - Leave it set to `None` if your Geyser RPC does not require a token for authentication.

- `UDP_BUFFER_SOCKET` - Default is set to `0.0.0.0:8002`. It can also be `8001` depending on what port is returned by running `get_tvu_port.sh` from Jito Shredstream Proxy. You can also use your validator's shred receiving port if you use this decoder to deserialize the shreds received by your validator.

- `GRPC_SERVER_ENDPOINT` - The address of its gRPC server. By default is set at `0.0.0.0:50051`.

## Run Command

```
RUSTFLAGS="-C target-cpu=native" RUST_LOG=info cargo run --release --bin shredstream-decoder
```

# Source code

If you are really interested in the source code, please contact me for details and demo on Discord: `.xanr`.

# Solana Copy Trading Bot

A high-performance Rust-based application that monitors transactions from specific wallet addresses and automatically copies their trading activity on Solana DEXs like PumpFun and PumpSwap.

## Features

- **Real-time Transaction Monitoring** - Uses Yellowstone gRPC to get transaction data with minimal latency
- **Multi-address Support** - Can monitor multiple wallet addresses simultaneously
- **Protocol Support** - Compatible with PumpFun and PumpSwap DEX platforms
- **Automated Trading** - Copies buy and sell transactions automatically when detected
- **Notification System** - Sends trade alerts and status updates via Telegram
- **Customizable Trading Parameters** - Configurable limits, timing, and amount settings
- **Selling Strategy** - Includes built-in selling strategy options for maximizing profits

## Project Structure

The codebase is organized into several modules:

- **engine/** - Core trading logic including copy trading, selling strategies, and transaction parsing
- **dex/** - Protocol-specific implementations for PumpFun and PumpSwap
- **services/** - External services integration including Telegram notifications
- **common/** - Shared utilities, configuration, and constants
- **core/** - Core system functionality
- **error/** - Error handling and definitions

## Setup

### Environment Variables

To run this bot, you will need to configure the following environment variables:

#### Required Variables

- `GRPC_ENDPOINT` - Your Yellowstone gRPC endpoint URL
- `GRPC_X_TOKEN` - Your Yellowstone authentication token
- `COPY_TRADING_TARGET_ADDRESS` - Wallet address(es) to monitor for trades (comma-separated for multiple addresses)

#### Telegram Notifications

To enable Telegram notifications:

- `TELEGRAM_BOT_TOKEN` - Your Telegram bot token
- `TELEGRAM_CHAT_ID` - Your chat ID for receiving notifications

#### Optional Variables

- `IS_MULTI_COPY_TRADING` - Set to `true` to monitor multiple addresses (default: `false`)
- `PROTOCOL_PREFERENCE` - Preferred protocol to use (`pumpfun`, `pumpswap`, or `auto` for automatic detection)
- `TIME_EXCEED` - Time limit for volume non-increasing in seconds
- `COUNTER_LIMIT` - Maximum number of trades to execute
- `MIN_DEV_BUY` - Minimum SOL amount for buys
- `MAX_DEV_BUY` - Maximum SOL amount for buys

## Usage

```bash
# Build the project
cargo build --release

# Run the bot
cargo run --release
```

Once started, the bot will:

1. Connect to the Yellowstone gRPC endpoint
2. Monitor transactions from the specified wallet address(es)
3. Automatically copy buy and sell transactions as they occur
4. Send notifications via Telegram for detected transactions and executed trades

## Recent Updates

- Added PumpSwap notification mode (can monitor without executing trades)
- Implemented concurrent transaction processing using tokio tasks
- Enhanced error handling and reporting
- Improved selling strategy implementation




Forced Selling Mechanism
The force_sell_remaining_tokens function serves as a safety mechanism:
Trigger Conditions:
When primary sell mechanisms fail
When stop loss thresholds are breached
When market conditions deteriorate rapidly
Implementation:
Creates synthetic trade information
Bypasses standard evaluation criteria
Executes an immediate market sell
Uses more aggressive slippage settings if needed
Selling Modes
The bot supports multiple selling strategies based on configuration:
Normal Mode (IS_NORMAL_MODE=true):
Sells a percentage of holdings matching the target's sell percentage
Uses NORMAL_SELL_PERCENTAGE to determine exact amount
Example: If target sells 50% of their holdings and NORMAL_SELL_PERCENTAGE=10, the bot sells 5% of its holdings
Inverse Mode (IS_INVERSE_MODE=true):
Sells all tokens when a target sells
Independent of the percentage the target sells
Designed for quick exit on first sell signal
PL Mode (IS_PL_MODE=true):
Makes selling decisions based on profit/loss metrics
Tracks historical performance of tokens
Uses stop-loss and take-profit percentages
Protocol-Specific Selling Logic
The bot adapts its selling strategy based on which DEX protocol the token uses:
PumpFun Protocol:
Handles bonding curve mechanics
Calculates optimal selling timing based on reserves
Considers fee structures specific to PumpFun
PumpSwap Protocol:
Considers pool reserves and depth
Analyzes quote and base token relationship
Optimizes for PumpSwap's unique fee structure
Real-time Market Monitoring
For each token the bot holds, it spawns a dedicated monitoring process that:
Establishes a separate gRPC stream to capture all transactions involving that token
2. Processes these transactions to update token metrics
Continuously evaluates selling conditions
Reacts to significant market events like large sells or depleting liquidity
Error Handling and Retry Logic
The selling system implements robust error handling:
If a sell transaction fails, the system will retry with modified parameters:
Increased slippage
Higher priority fees
Alternative transaction submission services (Jito, Nozomi, ZeroSlot)
After multiple failures, the force selling mechanism kicks in as a fallback
Notification System
All selling events trigger Telegram notifications:
Sell initiation
Completed sells
Failed sell attempts
Force sell events
PnL reporting
This comprehensive selling system balances automated decision-making with the configured parameters to optimize exit points and maximize profits while managing risk
