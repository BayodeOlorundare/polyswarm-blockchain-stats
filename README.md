![Polyswarm](https://bayodeolorundare.com/wp-content/uploads/2024/01/Polyswarm.png)

# Polyswarm Blockchain Stats

[*Dune dashboard can be accessed here*](https://dune.com/babylondon_204/polyswarm-blockchain-stats)

**Polyswarm Price**

```sql
SELECT price AS nct_price
FROM prices.usd
WHERE symbol='NCT'
ORDER BY minute DESC
LIMIT 1
```

**Polyswarm Market Capitalisation**

```sql
SELECT
  1885913075.851542181982426285 * nct_price AS market_cap
FROM (
  SELECT
    price AS nct_price
  FROM
    prices.usd
  WHERE
    symbol = 'NCT'
  ORDER BY
    minute DESC
  LIMIT 1
) AS derived_values;
```

**Polyswarm Price Line Chart**

```sql
SELECT date_trunc('day', minute) AS time
, AVG(price) AS price
FROM prices.usd
WHERE symbol='NCT'
GROUP BY 1
```

**Polyswarm NCT Weekly Holders**

```sql
SELECT 
  time,
  SUM(users) AS new_wallets,
  SUM(SUM(users)) OVER (ORDER BY time RANGE UNBOUNDED PRECEDING) AS cum_created_wallets
FROM (
  SELECT 
    date_trunc('week', creation_date) AS time,
    COUNT(users) AS users
  FROM (
    SELECT "to" AS users,
           MIN(call_block_time) AS creation_date
    FROM polyswarm_ethereum.NectarToken_call_transfer
    WHERE call_success
    GROUP BY "to"
  ) subquery
  GROUP BY date_trunc('week', creation_date)
) subquery2
GROUP BY time;
```

**Polyswarm Total Holders**

```sql
WITH transfers AS (
  SELECT
    MONTH,
    address,
    token_address,
    SUM(amount) AS amount
  FROM (
    SELECT
      DATE_TRUNC('month', evt_block_time) AS MONTH,
      "to" AS address,
      tr.contract_address AS token_address,
      value AS amount
    FROM erc20_ethereum.evt_Transfer AS tr
    WHERE
      contract_address = 0x9E46A38F5DaaBe8683E10793b06749EEF7D733d1
    UNION ALL
    SELECT
      DATE_TRUNC('month', evt_block_time) AS MONTH,
      "from" AS address,
      tr.contract_address AS token_address,
      -value AS amount
    FROM erc20_ethereum.evt_Transfer AS tr
    WHERE
      contract_address = 0x9E46A38F5DaaBe8683E10793b06749EEF7D733d1
  ) AS t
  GROUP BY
    1,
    2,
    3
), balances_with_gap_days AS (
  SELECT
    t.MONTH AS day,
    address,
    SUM(amount) OVER (PARTITION BY address ORDER BY t.MONTH) AS balance,
    LEAD(MONTH, 1, CURRENT_TIMESTAMP) OVER (PARTITION BY address ORDER BY t.MONTH) AS next_day
  FROM transfers AS t
), months AS (
  SELECT
    MONTH
  FROM UNNEST(SEQUENCE(
    TRY_CAST('2018-04-18' AS TIMESTAMP),
    CAST(TRY_CAST(DATE_TRUNC('month', CURRENT_TIMESTAMP) AS TIMESTAMP) AS TIMESTAMP),
    INTERVAL '1' month
  ) /* WARNING: Check out the docs for example of time series generation: https://dune.com/docs/query/syntax-differences/ */) AS _u(MONTH)
), balance_all_months AS (
  SELECT
    m.MONTH AS day,
    address,
    SUM(balance / POWER(10, 0)) AS balance
  FROM balances_with_gap_days AS b
  INNER JOIN months AS m
    ON b.day <= m.MONTH AND m.MONTH < b.next_day
  GROUP BY
    1,
    2
  ORDER BY
    1,
    2
)
SELECT
  b.day AS "Date",
  COUNT(address) AS "Holders",
  COUNT(address) - LAG(COUNT(address)) OVER (ORDER BY b.day) AS CHANGE
FROM balance_all_months AS b
WHERE
  balance > 0
GROUP BY
  1
ORDER BY
  1
```

**Total Polyswarm Wallets**

```sql
WITH transfers AS (
  SELECT
    evt_tx_hash AS tx_hash,
    tr."from" AS address,
    -tr.value AS amount,
    contract_address
  FROM erc20_ethereum.evt_Transfer AS tr
  WHERE
    contract_address = 0x9E46A38F5DaaBe8683E10793b06749EEF7D733d1
  UNION ALL
  SELECT
    evt_tx_hash AS tx_hash,
    tr."to" AS address,
    tr.value AS amount,
    contract_address
FROM erc20_ethereum.evt_Transfer AS tr
  WHERE
    contract_address = 0x9E46A38F5DaaBe8683E10793b06749EEF7D733d1
), transferAmounts AS (
  SELECT
    address,
    SUM(amount) / 1e18 AS poolholdings
  FROM transfers
  GROUP BY
    1
  ORDER BY
    2 DESC
)
SELECT
  COUNT(DISTINCT (
    address
  )) AS holders
FROM transferAmounts
WHERE
  poolholdings > 0
```

**Polyswarm Wallets with over 1,000 NCT **

```sql
WITH transfers AS (
  SELECT
    evt_tx_hash AS tx_hash,
    tr."from" AS address,
    -tr.value AS amount,
    contract_address
  FROM erc20_ethereum.evt_Transfer AS tr
  WHERE
    contract_address = 0x9E46A38F5DaaBe8683E10793b06749EEF7D733d1
  UNION ALL
  SELECT
    evt_tx_hash AS tx_hash,
    tr."to" AS address,
    tr.value AS amount,
    contract_address
FROM erc20_ethereum.evt_Transfer AS tr
  WHERE
    contract_address = 0x9E46A38F5DaaBe8683E10793b06749EEF7D733d1
), transferAmounts AS (
  SELECT
    address,
    SUM(amount) / 1e18 AS poolholdings
  FROM transfers
  GROUP BY
    1
  ORDER BY
    2 DESC
)
SELECT
  COUNT(DISTINCT (
    address
  )) AS holders
FROM transferAmounts
WHERE
  poolholdings > 1000
```

**Polyswarm Wallets with over 10,000 NCT**

```sql
WITH transfers AS (
  SELECT
    evt_tx_hash AS tx_hash,
    tr."from" AS address,
    -tr.value AS amount,
    contract_address
  FROM erc20_ethereum.evt_Transfer AS tr
  WHERE
    contract_address = 0x9E46A38F5DaaBe8683E10793b06749EEF7D733d1
  UNION ALL
  SELECT
    evt_tx_hash AS tx_hash,
    tr."to" AS address,
    tr.value AS amount,
    contract_address
FROM erc20_ethereum.evt_Transfer AS tr
  WHERE
    contract_address = 0x9E46A38F5DaaBe8683E10793b06749EEF7D733d1
), transferAmounts AS (
  SELECT
    address,
    SUM(amount) / 1e18 AS poolholdings
  FROM transfers
  GROUP BY
    1
  ORDER BY
    2 DESC
)
SELECT
  COUNT(DISTINCT (
    address
  )) AS holders
FROM transferAmounts
WHERE
  poolholdings > 10000
```

**Polyswarm Wallets with over 100,000 NCT**

```sql
WITH transfers AS (
  SELECT
    evt_tx_hash AS tx_hash,
    tr."from" AS address,
    -tr.value AS amount,
    contract_address
  FROM erc20_ethereum.evt_Transfer AS tr
  WHERE
    contract_address = 0x9E46A38F5DaaBe8683E10793b06749EEF7D733d1
  UNION ALL
  SELECT
    evt_tx_hash AS tx_hash,
    tr."to" AS address,
    tr.value AS amount,
    contract_address
FROM erc20_ethereum.evt_Transfer AS tr
  WHERE
    contract_address = 0x9E46A38F5DaaBe8683E10793b06749EEF7D733d1
), transferAmounts AS (
  SELECT
    address,
    SUM(amount) / 1e18 AS poolholdings
  FROM transfers
  GROUP BY
    1
  ORDER BY
    2 DESC
)
SELECT
  COUNT(DISTINCT (
    address
  )) AS holders
FROM transferAmounts
WHERE
  poolholdings > 100000
```

**Polyswarm Wallets with over 1,000,000 NCT**

```sql
WITH transfers AS (
  SELECT
    evt_tx_hash AS tx_hash,
    tr."from" AS address,
    -tr.value AS amount,
    contract_address
  FROM erc20_ethereum.evt_Transfer AS tr
  WHERE
    contract_address = 0x9E46A38F5DaaBe8683E10793b06749EEF7D733d1
  UNION ALL
  SELECT
    evt_tx_hash AS tx_hash,
    tr."to" AS address,
    tr.value AS amount,
    contract_address
FROM erc20_ethereum.evt_Transfer AS tr
  WHERE
    contract_address = 0x9E46A38F5DaaBe8683E10793b06749EEF7D733d1
), transferAmounts AS (
  SELECT
    address,
    SUM(amount) / 1e18 AS poolholdings
  FROM transfers
  GROUP BY
    1
  ORDER BY
    2 DESC
)
SELECT
  COUNT(DISTINCT (
    address
  )) AS holders
FROM transferAmounts
WHERE
  poolholdings > 1000000
```

**Polyswarm Wallets with over 10,000,000 NCT**

```sql
WITH transfers AS (
  SELECT
    evt_tx_hash AS tx_hash,
    tr."from" AS address,
    -tr.value AS amount,
    contract_address
  FROM erc20_ethereum.evt_Transfer AS tr
  WHERE
    contract_address = 0x9E46A38F5DaaBe8683E10793b06749EEF7D733d1
  UNION ALL
  SELECT
    evt_tx_hash AS tx_hash,
    tr."to" AS address,
    tr.value AS amount,
    contract_address
FROM erc20_ethereum.evt_Transfer AS tr
  WHERE
    contract_address = 0x9E46A38F5DaaBe8683E10793b06749EEF7D733d1
), transferAmounts AS (
  SELECT
    address,
    SUM(amount) / 1e18 AS poolholdings
  FROM transfers
  GROUP BY
    1
  ORDER BY
    2 DESC
)
SELECT
  COUNT(DISTINCT (
    address
  )) AS holders
FROM transferAmounts
WHERE
  poolholdings > 10000000
```

**Polyswarm (NCT) Whale Watch**

```sql
WITH transfers AS (
  SELECT
    evt_tx_hash AS tx_hash,
    tr."from" AS address,
    -tr.value AS amount,
    contract_address,
    evt_block_time
  FROM erc20_ethereum.evt_Transfer AS tr
  WHERE
    contract_address = 0x9E46A38F5DaaBe8683E10793b06749EEF7D733d1
  UNION ALL
  SELECT
    evt_tx_hash AS tx_hash,
    tr."to" AS address,
    tr.value AS amount,
    contract_address,
    evt_block_time
  FROM erc20_ethereum.evt_Transfer AS tr
  WHERE
    contract_address = 0x9E46A38F5DaaBe8683E10793b06749EEF7D733d1
), transferAmounts AS (
  SELECT
    address,
    TRY_CAST(SUM(amount) / 1e18 AS DECIMAL(18, 2)) AS poolholdings /* Adding commas to the balance */
  FROM transfers
  GROUP BY
    1
), dailyChange AS (
  SELECT
    address,
    TRY_CAST(SUM(
      CASE
        WHEN evt_block_time >= CURRENT_TIMESTAMP - INTERVAL '7' day
        THEN amount
        ELSE 0
      END
    ) / 1e18 AS DECIMAL(18, 2)) AS daily_change /* Adding commas to the daily_change */
  FROM transfers
  GROUP BY
    1
)
SELECT
  t.address,
  CASE
    WHEN t.address = 0xC03498CFcdb97AA2CF8e5939d7297C4496f4ee91 THEN 'Bithumb'
    WHEN t.address = 0xA9D1e08C7793af67e9d92fe308d5697FB81d3E43 THEN 'Coinbase10'
    WHEN t.address = 0x0D0707963952f2fBA59dD06f2b425ace40b492Fe THEN 'Gate.io'
    WHEN t.address = 0x75e89d5979E4f6Fba9F97c104c2F0AFB3F1dcB88 THEN 'MEXC'
    WHEN t.address = 0x39fB0dCd13945B835d47410Ae0DE7181D3edf270 THEN 'Bitmart (Hacker)'
    WHEN t.address = 0x4bb7D80282F5E0616705D7f832AcFC59F89F7091 THEN 'Bitmart (Hacker) 2'
    WHEN t.address = 0x18709E89BD403F470088aBDAcEbE86CC60dda12e THEN 'Huobi'
    WHEN t.address = 0x2a0c0DBEcC7E4D658f48E01e3fA353F44050c208 THEN 'Idex'
    WHEN t.address = 0x05aaa0053Fa5c28e8C558d4c648Cc129bEa45018 THEN 'Uniswap V3'
    WHEN t.address = 0xB2cc3cDd53fC9A1AEAf3A68Edeba2736238DDC5D THEN 'TopBTC'
    WHEN t.address = 0x8d12A197cB00D4747a1fe03395095ce2A5CC6819 THEN 'EtherDelta2'
    WHEN t.address = 0xb154256024041e3011c8024874e698b69a2081b3 THEN 'Uniswap V2'
    WHEN t.address = 0xe135eCa22512f5281b06d4F4242a43B84EEeC710 THEN 'TopBTC: Deployer'
    WHEN t.address = 0xFBb1b73C4f0BDa4f67dcA266ce6Ef42f520fBB98 THEN 'Bittrex'
    WHEN t.address = 0x3CC936b795A188F0e246cBB2D74C5Bd190aeCF18 THEN 'MEXC 3'
    WHEN t.address = 0x6025D96932D378BE7D0A46343b437678A126eCCa THEN 'HitBTC'
  END AS name,
  TRY_CAST(t.poolholdings AS VARCHAR(18)) AS balance,
  COALESCE(TRY_CAST(dc.daily_change AS VARCHAR(18)), '0') AS daily_change
FROM
  transferAmounts AS t
LEFT JOIN
  dailyChange AS dc ON t.address = dc.address
WHERE
  t.poolholdings > 10000000
ORDER BY
  t.poolholdings DESC;
```

**Polyswarm Exchange Holdings Tracker**

```sql
WITH transfers AS (
  SELECT
    evt_tx_hash AS tx_hash,
    tr."from" AS address,
    -tr.value AS amount,
    contract_address,
    evt_block_time
  FROM erc20_ethereum.evt_Transfer AS tr
  WHERE
    contract_address = 0x9E46A38F5DaaBe8683E10793b06749EEF7D733d1
  UNION ALL
  SELECT
    evt_tx_hash AS tx_hash,
    tr."to" AS address,
    tr.value AS amount,
    contract_address,
    evt_block_time
  FROM erc20_ethereum.evt_Transfer AS tr
  WHERE
    contract_address = 0x9E46A38F5DaaBe8683E10793b06749EEF7D733d1 AND (
    tr."to" IN (0xC03498CFcdb97AA2CF8e5939d7297C4496f4ee91, 0xA9D1e08C7793af67e9d92fe308d5697FB81d3E43, 0x0D0707963952f2fBA59dD06f2b425ace40b492Fe, 0x75e89d5979E4f6Fba9F97c104c2F0AFB3F1dcB88, 0x39fB0dCd13945B835d47410Ae0DE7181D3edf270, 0x4bb7D80282F5E0616705D7f832AcFC59F89F7091, 0x18709E89BD403F470088aBDAcEbE86CC60dda12e, 0x2a0c0DBEcC7E4D658f48E01e3fA353F44050c208, 0x05aaa0053Fa5c28e8C558d4c648Cc129bEa45018, 0xB2cc3cDd53fC9A1AEAf3A68Edeba2736238DDC5D, 0x8d12A197cB00D4747a1fe03395095ce2A5CC6819, 0xb154256024041e3011c8024874e698b69a2081b3,0xe135eCa22512f5281b06d4F4242a43B84EEeC710, 0xFBb1b73C4f0BDa4f67dcA266ce6Ef42f520fBB98, 0x3CC936b795A188F0e246cBB2D74C5Bd190aeCF18, 0x6025D96932D378BE7D0A46343b437678A126eCCa))
), transferAmounts AS (
  SELECT
    address,
    TRY_CAST(SUM(amount) / 1e18 AS DECIMAL(18, 2)) AS poolholdings /* Adding commas to the balance */
  FROM transfers
  GROUP BY
    1
), dailyChange AS (
  SELECT
    address,
    TRY_CAST(SUM(
      CASE
        WHEN evt_block_time >= CURRENT_TIMESTAMP - INTERVAL '7' day
        THEN amount
        ELSE 0
      END
    ) / 1e18 AS DECIMAL(18, 2)) AS daily_change /* Adding commas to the daily_change */
  FROM transfers
  GROUP BY
    1
)
SELECT
  t.address,
  CASE
    WHEN t.address = 0xC03498CFcdb97AA2CF8e5939d7297C4496f4ee91 THEN 'Bithumb'
    WHEN t.address = 0xA9D1e08C7793af67e9d92fe308d5697FB81d3E43 THEN 'Coinbase10'
    WHEN t.address = 0x0D0707963952f2fBA59dD06f2b425ace40b492Fe THEN 'Gate.io'
    WHEN t.address = 0x75e89d5979E4f6Fba9F97c104c2F0AFB3F1dcB88 THEN 'MEXC'
    WHEN t.address = 0x39fB0dCd13945B835d47410Ae0DE7181D3edf270 THEN 'Bitmart (Hacker)'
    WHEN t.address = 0x4bb7D80282F5E0616705D7f832AcFC59F89F7091 THEN 'Bitmart (Hacker) 2'
    WHEN t.address = 0x18709E89BD403F470088aBDAcEbE86CC60dda12e THEN 'Huobi'
    WHEN t.address = 0x2a0c0DBEcC7E4D658f48E01e3fA353F44050c208 THEN 'Idex'
    WHEN t.address = 0x05aaa0053Fa5c28e8C558d4c648Cc129bEa45018 THEN 'Uniswap V3'
    WHEN t.address = 0xB2cc3cDd53fC9A1AEAf3A68Edeba2736238DDC5D THEN 'TopBTC'
    WHEN t.address = 0x8d12A197cB00D4747a1fe03395095ce2A5CC6819 THEN 'EtherDelta2'
    WHEN t.address = 0xb154256024041e3011c8024874e698b69a2081b3 THEN 'Uniswap V2'
    WHEN t.address = 0xe135eCa22512f5281b06d4F4242a43B84EEeC710 THEN 'TopBTC: Deployer'
    WHEN t.address = 0xFBb1b73C4f0BDa4f67dcA266ce6Ef42f520fBB98 THEN 'Bittrex'
    WHEN t.address = 0x3CC936b795A188F0e246cBB2D74C5Bd190aeCF18 THEN 'MEXC 3'
    WHEN t.address = 0x6025D96932D378BE7D0A46343b437678A126eCCa THEN 'HitBTC'
  END AS name,
  TRY_CAST(t.poolholdings AS VARCHAR(18)) AS balance,
  COALESCE(TRY_CAST(dc.daily_change AS VARCHAR(18)), '0') AS daily_change
FROM
  transferAmounts AS t
LEFT JOIN
  dailyChange AS dc ON t.address = dc.address
WHERE
  t.poolholdings > 0
ORDER BY
  t.poolholdings DESC;
```

**Total NCT Transactions**

```sql
SELECT
  DATE_TRUNC('day', block_time),
  COUNT(DISTINCT (
    "hash"
  ))
FROM ethereum.transactions
WHERE
  "to" = 0x9E46A38F5DaaBe8683E10793b06749EEF7D733d1 AND success = TRUE
GROUP BY
  1
```
