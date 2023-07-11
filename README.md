-- dunesql_alpha_deprecated
-- This query has incompatible functions with our new data types and is running against the
-- Dune SQL alpha deprecated engine (with old data types), which will stop working on March 30th 2023.
-- Please remove the comment in the first line and migrate to using new compatible Dune SQL functions found in
-- our documentation page so you can benefit from better usability, continued usage, and up to 40% faster query speeds.
with 
address(address) as (
    values
    (0x2faf487a4414fe77e2327f0bf4ae2a264a776ad2), 
    (0xc098b2a3aa256d2140208c3de6543aaef5cd3a94)
), token_price_dec as (
    select date_trunc('day', minute) as time, symbol, contract_address, decimals, avg(price) as avg_price
    from prices.usd
    where minute >= cast('2022-11-06' as TIMESTAMP) and minute < cast('2022-11-07' as TIMESTAMP)
        and blockchain = 'ethereum'
        and contract_address = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2
        and symbol = 'WETH'
    group by 1, 2, 3, 4
), eth_in as (
    select date_trunc('day', block_time) as time, 'in' as category, sum( cast(value as double) / pow(10, p.decimals)) as raw_amt, sum( cast(value as double) / pow(10, p.decimals) * p.avg_price) as usd_amt
    from ethereum.traces t
    left join token_price_dec p on p.time = date_trunc('day', block_time)
    where t.block_time >= cast('2022-11-06' as TIMESTAMP) and t.block_time < cast('2022-11-07' as TIMESTAMP)
        and t."to" in (select address from address) 
    group by 1, 2
), eth_out as (
    select date_trunc('day', block_time) as time, 'out' as category, -1 * sum( cast(value as double) / pow(10, p.decimals)) as raw_amt, -1 * sum( cast(value as double) / pow(10, p.decimals) * p.avg_price) as usd_amt
    from ethereum.traces t
    left join token_price_dec p on date_trunc('day', block_time) = p.time
    where t.block_time >= cast('2022-11-06' as TIMESTAMP) and t.block_time < cast('2022-11-07' as TIMESTAMP)
        and t."from" in (select address from address) 
    group by 1, 2
)
select sum(usd_amt) as usd_netflow
from (
    select time, category, raw_amt, usd_amt
    from eth_in
    union all 
    select time, category, raw_amt, usd_amt
    from eth_out
)
