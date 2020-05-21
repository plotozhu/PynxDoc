# 说明
在交易池里轮转交易。 仅仅保留最新的交易，放弃超过一定时间的交易。 被放弃的交易将被屏蔽一段时间，以名被再次导入。

# PoolRotator
PoolRotator 负责让交易池里保持最新的交易。 在交易池里占用太多时间的交易将会被删除，并且会被屏蔽一段时间不允许再次进入交易池
```rust
pub struct PoolRotator<Hash> {
    /// 交易被放弃后，禁止再加入的时间
    ban_time: Duration,
    /// 当前被阻止的交易
    banned_until: RwLock<HashMap<Hash, Instant>>,
}
```

## 是否被屏蔽
`pub fn is_banned(&self, hash: &Hash) -> bool {`

## 屏蔽某些交易
`pub fn ban(&self, now: &Instant, hashes: impl IntoIterator<Item=Hash>) {`

## 检测是否过期，过期就屏蔽
`pub fn ban_if_stale<Ex>(&self, now: &Instant, current_block: u64, xt: &Transaction<Hash, Ex>) -> bool {`

## 清除屏蔽时间
`pub fn clear_timeouts(&self, now: &Instant) {`
