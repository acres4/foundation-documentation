# Foundation Event Replay API

## Connection

Users connect to the event stream replay API over a websocket located at `wss://<property>.kailabor.com/nex7/sas-reader/ws/historical-event-streamer` where `<property>` is the subdomain for the casino running Foundation. Clients enter parameters determining which events to stream as URL parameters following `?`. The API interprets the parameters as follows:

| Field | Description |
| --- | ------- |
| start | The time from which to start streaming data, expressed as nanoseconds since the beginning of the Unix epoch |
| end | The time at which to stop streaming data, expressed as nanoseconds since the beginning of the Unix epoch |
| limit | Limit the total number of packets to process before sending to the client |
| raw-json | Send data to the client as a flattened packet described by the data dictionary below. The value should simply be set to `true` |
| sas_codes | A comma delimited list of real time SAS exception codes. If set, the server will only stream data matching these SAS exception codes to the client. The full list of real time SAS exception codes can be found in Slot Accounting System Protocol Version 6.02 Appendix A Table A-1. Enter codes in decimal notation. |
| asset_number | A comma delimited list of asset numbers. If set, the server will send data matching these asset numbers. Note that if `sas_serial_number` is also set, the server will apply an `OR` filter to the data |
| sas_serial_number | A comma delimited list of SAS serial numbers. If set, the server will send data matching these SAS serial numbers. Note that if `asset_number` is also set, the server will apply an `OR` filter to the data |
| start_idx | Sets the starting `idx` field from which to read data. Acres4 uses this field to apply a global ordering to all data packets in the stream |
| end_idx | Sets the ending `idx` field to which to read data. Acres4 uses this field to apply a global ordering to all data packets in the stream |

If using `start` and `end` filters, leave `start_idx` and `end_idx` fields blank. Likewise, if using `start_idx` and `end_idx` fields, leave `start` and `end` fields blank. Users should also refrain from mixing the two filter sets. Because indexes are sequential, any `limit` set on a query bounded by `start_idx` and `end_idx` is ignored. Use these fields to bound your query instead. An example URL for reading data is as follows:

`wss://mycasino.kailabor.com/nex7/sas-reader/ws/historical-event-streamer?start=1561705200000000000&end=1562184823000000000&limit=50000&raw-json=true`

Users must upgrade all requests to a websocket to avoid a `400` error from the server.

## Performance

Acres 4 have tested the websocket connection with limits up to 1,000,000 packets. This limit is placed on the number of unified event packets processed through the server and may not reflect the number of flattened data packets. Some event packets do not yield a flattened packet and others yield multiple packets. Unified event packets that do not yield a flattened packet are used by Acres 4 for internal purposes only. Processing the same unified event packets yields the same set of flattened packets across multiple trials.

Testing shows the server writes packets at a rate of approximate 7,400 packets/second or 135µs/packet. Slower clients may not receive all packets the server has sent. Acres 4 tested client performance using an API implementation written in [Go](https://golang.org) that can introduce arbitrary delays into processing. Testing shows that the following maximum limits lead to consistent results based on client processing speed. For convenience, we have listed processing speed both in terms of processing time per packet and packets per second.

| Processing Time / Packet | Packets / Second | Maximum Query Limit |
|:---:|:-----:|:-----:|
| 135µs | 7,400 | 1,000,000 |
| 185µs | 5,400 | 500,000 |
| 250µs | 4,000 | 250,000 |
| 490µs | 2,000 | 100,000 |
| 6ms | 165 | 50,000 |

## Data Structure

The server sends flattened data packets to the server in JSON form. Flattened packets take the following structure:

```
{
    "type": "string",
    "sas_serial_number": "string",
    "player_card_number": "string",
    "asset_number": "string",
    "host_id_player": "string",
    "idx": int,
    "cache_time": int64,
    "data" : map{...},
    "meters" : map{...}
}
```

An example packet might take the following form:

```
{
    "type":"SAS_EVENT_CODE_7E_GAME_HAS_STARTED",
    "sas_serial_number":"80393",
    "player_card_number":"",
    "asset_number":"30503",
    "host_id_player":"49000713",
    "idx":55903787,
    "cache_time":1561852795766,
    "data":{
        "CreditsWagered":120,
        "HostIDPlayer":"49000713",
        "MIBTime":1561852793880,
        "ProgressiveGroup":0,
        "SASEvent":126,
        "SASPoll":255,
        "TotalCoinInMeter":50522580,
        "WagerType":65
    },
    "meters":{
        "CoinIn":50522460,
        "CoinOut":44265651,
        "CurrentCredits":17825,
        "GamesPlayed":326291,
        "GamesWon":89114,
        "HandpayCancelCredit":0,
        "Jackpot":1064925
    }
}
```

Fields are defined in the data dictionary below for all packet types

## Data Dictionary

### PlayerInfo

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Data

| field | description |
|---|-----|
| CacheTime | The time at which the Acres 4 global event collector collected this event |
| FirstName | The first name of the player |
| LastLocation | The last location of the player |
| MembershipTierStr | The string representation of the membership tier level for this player |
| CardNumber | The card number for this player |
| Gender | The gender for this player |
| HostClubNumber | The player number as reported by the Bally system |
| LastEventTime | The last time at which Acres 4 saw an event for this player |
| LastName | The last name of this player |
| LoyaltyTierStr | Same as MembershipTierStr |
| Theo | The theoretical win for this player |
| Banned | If true, the player is banned from this casino |
| BirthDate | The date of birth for this player |
| Crc | Used for internal diagnostics by Acres 4 |
| HostIdPlayer | The unique identifier for this player as reported by the Bally system |
| ZipCode | The zip code for this player |
| ZipExt | The zip code extension for the player |

### SAS_POLL_FF_REAL_TIME_EXCEPTION

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| RecallIndex | The recall index for the game being recalled |
| DenomCode | Indicates the denomination code for the bill accepted |
| Bills | Indicates the number of bills accepted |
| MultipliedWin | The multiplied win value |
| TotalCoinInMeter | The value of the total coin in meter at the start of the game |
| Card | The index of the card being held |
| Sig | The ROM signature |
| WagerType | The wager type |
| ProgressiveGroup | The progressive group for the game |
| GameWin | The amount won from the game |
| Multiplier | The event bonus pay multiplier |
| PhysicalStop | The position in which the reel stopped |
| GameNumber | The number of the game being recalled or selected. Use the SASEvent field to determine the meaning of this key |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |
| Seed | The seed value on the ROM signature |
| CountryCode | Indicates the country code for the bill accepted |
| ReelNum | The number of the reel for which a stop is being reported |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| TaxStatus | The tax status of the bonus paid |
| BonusWin | The bonus win amount |
| CreditsWagered | The number of credits wagered on this game |

### SAS_EVENT_CODE_9B_POWER_OFF_DROP_DOOR_ACCESS

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_89_COIN_CREDIT_WAGERED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |

### SAS_POLL_51_SEND_TOTAL_NUMBER_OF_GAMES_IMPLEMENTED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| NumberOfGames | Total games implemented by this gaming machine |

### SAS_POLL_48_SEND_LAST_ACCEPTED_BILL_INFORMATION

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| Bills | Indicates the number of bills accepted |
| Country | The country code for the bills accepted |
| Denom | The denom of the game as reported by the Bally system |

### SAS_EVENT_CODE_60_PRINTER_COMMUNICATION_ERROR

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_POLL_A8_ENABLE_JACKPOT_HANDPAY_RESET_METHOD

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| SASPoll | The code sent the gaming machine to request information |
| AckCode | Used by Acres 4 to implement the SAS protocol |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |

### SendGamingMachineIDAndInformation

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Value of the current credit meter as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid credits canceled as reported by the gaming machine |
| Jackpot | Total value of all jackpots as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| GameOptions | The game options |
| MaxBet | The maximum bet |
| PaytableId | The paytable ID |
| ProgressiveGroup | The progressive group as reported by the gaming machine |
| AdditionalId | Additional ID information |
| BasePercentage | The base percentage |
| Denomination | The denomination |
| GameId | The game ID |

### SAS_EVENT_CODE_72_CHANGE_LAMP_OFF

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_4F_BILL_ACCEPTED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| Bills | Indicates the number of bills accepted |
| CountryCode | Indicates the country code for the bill accepted |
| DenomCode | Indicates the denomination code for the bill accepted |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_19_CASHBOX_DOOR_WAS_OPENED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| SASPoll | The code sent the gaming machine to request information |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |

### SendPhysicalReelStopInformation

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| Jackpot | Total value of all jackpots as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Value of the current credit meter as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid credits canceled as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| PhysicalReelStops | The current physical reel stops |

### SAS_EVENT_CODE_53_NO_PROGRESSIVE_INFORMATION_HAS_BEEN_RECEIVED_FOR_5_SECONDS

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_16_CARD_CAGE_WAS_CLOSED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_POLL_1F_SEND_GAMING_MACHINE_ID_AND_INFORMATION

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| Denomination | The denomination |
| GameId | The game ID |
| GameOptions | The game options |
| MaxBet | The maximum bet |
| PaytableId | The paytable ID |
| ProgressiveGroup | The progressive group for the game |
| AdditionalId | Additional ID information |
| BasePercentage | The base percentage |

### SAS_EVENT_CODE_7B_BILL_VALIDATOR

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| SASPoll | The code sent the gaming machine to request information |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |

### SAS_EVENT_CODE_3D_A_CASH_OUT_TICKET_HAS_BEEN_PRINTED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |

### SAS_EVENT_CODE_29_BILL_ACCEPTOR_HARDWARE_FAILURE

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| SASPoll | The code sent the gaming machine to request information |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |

### SAS_EVENT_CODE_83_DISPLAY_METERS_OR_ATTENDANT_MENU_HAS_BEEN_EXITED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |

### SAS_EVENT_CODE_74_PRINTER_PAPER_LOW

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_12_SLOT_DOOR_WAS_CLOSED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| SASPoll | The code sent the gaming machine to request information |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |

### SAS_EVENT_CODE_88_REEL_N_HAS_STOPPED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| PhysicalStop | The position in which the reel stopped |
| ReelNum | The number of the reel for which a stop is being reported |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_7E_GAME_HAS_STARTED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| CreditsWagered | The number of credits wagered on this game |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| ProgressiveGroup | The progressive group for the game |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |
| TotalCoinInMeter | The value of the total coin in meter at the start of the game |
| WagerType | The wager type |

### SAS_EVENT_CODE_61_PRINTER_PAPER_OUT_ERROR

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_42_REEL_2_TILT

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |

### SAS_EVENT_CODE_8A_GAME_RECALL_ENTRY_HAS_BEEN_DISPLAYED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| GameNumber | The number of the game being recalled or selected. Use the SASEvent field to determine the meaning of this key |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| RecallIndex | The recall index for the game being recalled |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### MetersGo

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Data

| field | description |
|---|-----|
| TOTAL_TICKET_IN_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| DEBIT_TRANSFERS_TO_TICKET_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| REGULAR_CASHABLE_TICKET_IN_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| RESERVED_FOR_FUTURE_USE_9B | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| RESERVED_FOR_FUTURE_USE_B7 | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_1000000_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_REGULAR_CASHABLE_TICKET_IN_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_RESTRICTED_AMOUNT_PLAYED_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_SAS_RESTRICTED_TICKET_IN_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| BONUS_NONRESTRICTED_TRANSFERS_TO_GAMING_MACHINE_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| REGULAR_CASHABLE_TICKET_OUT_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| SPECIAL_METERS_POWER_RESET | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_CASHABLE_TICKET_OUT_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| VALIDATED_NO_RECEIPT_JACKPOT_HANDPAY_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| VALIDATED_NO_RECEIPT_JACKPOT_HANDPAY_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_50000_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_500_BILLS_TO_DROP | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_50_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_SAS_CASHABLE_TICKET_IN_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| ELECTRONIC_NONRESTRICTED_PROMOTIONAL_TRANSFERS_TO_GAMING_MACHINE_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| ELECTRONIC_RESTRICTED_PROMOTIONAL_TRANSFERS_TO_GAMING_MACHINE_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| IN_HOUSE_TRANSFERS_TO_HOST_THAT_INCLUDED_CASHABLE_AMOUNTS_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| SPECIAL_METERS_TOTAL_CREDIT_VALUE_OF_BILLS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_SAS_RESTRICTED_TICKET_IN_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_VALUE_OF_BILLS_CURRENTLY_IN_THE_STACKER_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| DEBIT_TRANSFERS_TO_TICKET_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| IN_HOUSE_RESTRICTED_TRANSFERS_TO_GAMING_MACHINE_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| IN_HOUSE_TRANSFERS_TO_GAMING_MACHINE_THAT_INCLUDED_RESTRICTED_AMOUNTS_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_MACHINE_PAID_PROGRESSIVE_WIN_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| REGULAR_CASHABLE_TICKET_OUT_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| RESERVED_FOR_FUTURE_USE_9E | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| RESERVED_FOR_FUTURE_USE_B4 | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_ATTENDANT_PAID_PROGRESSIVE_WIN_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_20000_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_25_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_500_BILLS_DISPENSED_FROM_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| IN_HOUSE_TRANSFERS_TO_GAMING_MACHINE_THAT_INCLUDED_CASHABLE_AMOUNTS_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| IN_HOUSE_TRANSFERS_TO_GAMING_MACHINE_THAT_INCLUDED_NONRESTRICTED_AMOUNTS_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_CREDITS_FROM_BILLS_TO_DROP | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_1000_BILLS_DISPENSED_FROM_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_20_BILLS_DIVERTED_TO_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_SAS_RESTRICTED_TICKET_OUT_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| GAMES_PLAYED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| RESERVED_FOR_FUTURE_USE_99 | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| SPECIAL_METERS_TRUE_COIN_IN | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_ATTENDANT_PAID_EXTERNAL_BONUS_WIN_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_200_BILLS_DIVERTED_TO_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| VALIDATED_NO_RECEIPT_CANCELLED_CREDIT_HANDPAY_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| VALIDATED_RECEIPT_CANCELLED_CREDIT_HANDPAY_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| IN_HOUSE_CASHABLE_TRANSFERS_TO_HOST_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| NUMBER_OF_BILLS_CURRENTLY_IN_THE_STACKER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| SPECIAL_METERS_TOURNAMENT_CREDIT_WON | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_1000_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_20_BILLS_DISPENSED_FROM_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| CURRENT_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| RESERVED_FOR_FUTURE_USE_B3 | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| SESSIONS_PLAYED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_CREDITS_FROM_BILLS_DIVERTED_TO_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_50_BILLS_DISPENSED_FROM_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_SAS_CASHABLE_TICKET_IN_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_TICKET_OUT_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| RESERVED_FOR_FUTURE_USE_9F | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| RESTRICTED_TICKET_OUT_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_100_BILLS_TO_DROP | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_20_BILLS_TO_DROP | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_RESTRICTED_PROMOTIONAL_TICKET_OUT_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| SPECIAL_METERS_TRUE_COIN_OUT | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_HAND_PAID_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_200_BILLS_TO_DROP | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_2_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_CREDITS_FROM_COIN_ACCEPTOR | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_JACKPOT_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_10_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_2_BILLS_TO_DROP | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| GAMES_LOST | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| GAMES_WON | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| IN_HOUSE_RESTRICTED_TRANSFERS_TO_TICKET_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| RESERVED_FOR_FUTURE_USE_9C | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_100_BILLS_DIVERTED_TO_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_1_BILLS_DISPENSED_FROM_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| IN_HOUSE_TRANSFERS_TO_HOST_THAT_INCLUDED_RESTRICTED_AMOUNTS_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_COIN_IN_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_HAND_PAID_CANCELLED_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NONRESTRICTED_AMOUNT_PLAYED_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_ELECTRONIC_TRANSFERS_TO_HOST_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_10_BILLS_TO_DROP | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_SAS_CASHABLE_TICKET_OUT_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_CREDITS_FROM_EXTERNAL_COIN_ACCEPTOR | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_MACHINE_PAID_EXTERNAL_BONUS_WIN_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_20_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_5_BILLS_DIVERTED_TO_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| IN_HOUSE_TRANSFERS_TO_HOST_THAT_INCLUDED_NONRESTRICTED_AMOUNTS_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| RESERVED_FOR_FUTURE_USE_B5 | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| SPECIAL_METERS_CURRENT_HOPPER_LEVEL | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_CANCELLED_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| VALIDATED_RECEIPT_JACKPOT_HANDPAY_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_10000_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_10_BILLS_DISPENSED_FROM_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_250_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_500000_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| ELECTRONIC_DEBIT_TRANSFERS_TO_GAMING_MACHINE_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| IN_HOUSE_NONRESTRICTED_TRANSFERS_TO_GAMING_MACHINE_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| RESTRICTED_TICKET_OUT_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_CREDITS_FROM_BILLS_DISPENSED_FROM_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_500_BILLS_DIVERTED_TO_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_50_BILLS_TO_DROP | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_RESTRICTED_PROMOTIONAL_TICKET_IN_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_CREDITS_FROM_COINS_TO_DROP | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_100000_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_10_BILLS_DIVERTED_TO_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_5_BILLS_TO_DROP | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| BONUS_TRANSFERS_TO_GAMING_MACHINE_THAT_INCLUDED_CASHABLE_AMOUNTS_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| RESERVED_FOR_FUTURE_USE_9A | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| RESTRICTED_TICKET_IN_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_REGULAR_CASHABLE_TICKET_IN_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| SPECIAL_METERS_TOURNAMENT_GAMES_WON | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| WEIGHTED_AVERAGE_THEORETICAL_PAYBACK_PERCENTAGE_IN_HUNDREDTHS_OF_A_PERCENT | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| SPECIAL_METERS_TOURNAMENT_GAMES_PLAYED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| BONUS_CASHABLE_TRANSFERS_TO_GAMING_MACHINE_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| CURRENT_RESTRICTED_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| RESERVED_FOR_FUTURE_USE_B2 | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| SPECIAL_METERS_TOTAL_DOLLAR_VALUE_OF_BILLS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_100_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_1_BILLS_DIVERTED_TO_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| IN_HOUSE_NONRESTRICTED_TRANSFERS_TO_HOST_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| RESERVED_FOR_FUTURE_USE_B6 | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| RESTRICTED_TICKET_IN_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NONRESTRICTED_PROMOTIONAL_TICKET_IN_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_DROP_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_25000_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_5_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| DEBIT_TICKET_OUT_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| DEBIT_TRANSFERS_TO_GAMING_MACHINE_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| ELECTRONIC_REGULAR_CASHABLE_TRANSFERS_TO_HOST_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| IN_HOUSE_CASHABLE_TRANSFERS_TO_GAMING_MACHINE_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NONRESTRICTED_PROMOTIONAL_TICKET_IN_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_100_BILLS_DISPENSED_FROM_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| ELECTRONIC_NONRESTRICTED_PROMOTIONAL_TRANSFERS_TO_HOST_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| IN_HOUSE_RESTRICTED_TRANSFERS_TO_TICKET_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| SPECIAL_METERS_LEGACY_BONUS_DEDUCTIBLE_METER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_ATTENDANT_PAID_PAYTABLE_WIN_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_RESTRICTED_PROMOTIONAL_TICKET_IN_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| VALIDATED_RECEIPT_JACKPOT_HANDPAY_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| REGULAR_CASHABLE_TICKET_IN_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_CASHABLE_TICKET_IN_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_ELECTRONIC_TRANSFERS_TO_GAMING_MACHINE_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_5000_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_500_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_SAS_RESTRICTED_TICKET_OUT_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_WON_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| SPECIAL_METERS_LEGACY_BONUS_WAGER_MATCH | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_1000_BILLS_TO_DROP | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_2500_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_2_BILLS_DISPENSED_FROM_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| DEBIT_TICKET_OUT_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_MACHINE_PAID_PAYTABLE_WIN_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_1_BILLS_TO_DROP | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_250000_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| NONRESTRICTED_TICKET_IN_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_1000_BILLS_DIVERTED_TO_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_200_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_2_BILLS_DIVERTED_TO_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| BONUS_TRANSFERS_TO_GAMING_MACHINE_THAT_INCLUDED_NONRESTRICTED_AMOUNTS_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| IN_HOUSE_CASHABLE_TRANSFERS_TO_TICKET_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| NONRESTRICTED_TICKET_IN_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_CASHABLE_TICKET_OUT_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_200000_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_5_BILLS_DISPENSED_FROM_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| GAMES_SINCE_LAST_POWER_RESET | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| IN_HOUSE_CASHABLE_TRANSFERS_TO_TICKET_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| SPECIAL_METERS_LEGACY_BONUS_NON_DEDUCTIBLE_METER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_1_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_2000_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_RESTRICTED_PROMOTIONAL_TICKET_OUT_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| ELECTRONIC_REGULAR_CASHABLE_TRANSFERS_TO_GAMING_MACHINE_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| GAMES_SINCE_SLOT_DOOR_CLOSURE | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| SPECIAL_METERS_SLOT_DOOR_OPEN | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| SPECIAL_METERS_TOURNAMENT_CREDIT_WAGERED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_SAS_CASHABLE_TICKET_OUT_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| VALIDATED_RECEIPT_CANCELLED_CREDIT_HANDPAY_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| ELECTRONIC_RESTRICTED_PROMOTIONAL_TRANSFERS_TO_HOST_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| RESERVED_FOR_FUTURE_USE_9D | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_CREDITS_FROM_BILLS_ACCEPTED | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_CREDITS_PAID_FROM_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_50_BILLS_DIVERTED_TO_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| VALIDATED_NO_RECEIPT_CANCELLED_CREDIT_HANDPAY_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| DEBIT_TRANSFERS_TO_GAMING_MACHINE_QUANTITY | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| IN_HOUSE_RESTRICTED_TRANSFERS_TO_HOST_CENTS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_COIN_OUT_CREDITS | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |
| TOTAL_NUMBER_OF_200_BILLS_DISPENSED_FROM_HOPPER | See Slot Accounting System Protocol Version 6.02 Appendix C Table C-7 |

### SAS_POLL_56_SAS_SEND_ENABLED_GAME_NUMBERS

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| EnabledGameNumbers | The numbers for the games that are enabled |
| Length | The length of the packet |
| NumberOfGames | Total games implemented by this gaming machine |

### SAS_EVENT_CODE_28_BILL_JAM

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_17_AC_POWER_WAS_APPLIED_TO_GAMING_MACHINE

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_71_CHANGE_LAMP_ON

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| SASPoll | The code sent the gaming machine to request information |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |

### SAS_EVENT_CODE_1E_BELLY_DOOR_WAS_CLOSED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |

### SAS_POLL_A0_SEND_ENABLED_FEATURES

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| Reserved | Reserved |
| Features | The featrues that are enabled |
| GameNumber | The number of the game being recalled or selected. Use the SASEvent field to determine the meaning of this key |

### SAS_EVENT_CODE_1B_CASHBOX_WAS_REMOVED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_14_DROP_DOOR_WAS_CLOSED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_32_CMOS_RAM_ERROR

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Data

| field | description |
|---|-----|
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |

### SAS_EVENT_CODE_1C_CASHBOX_WAS_INSTALLED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_51_HANDPAY_IS_PENDING

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_POLL_55_SEND_SELECTED_GAME_NUMBER

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| SelectedGameNumber | Indicates which game is being selected |

### SAS_POLL_1B_SEND_HANDPAY_INFORMATION

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| Unused | N/A |
| Amount | The amount to be paid by this handpay |
| Level | The handpay level as reported by the gaming machine |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| PartialPayAmount | The partial pay amount |
| ProgressiveGroup | The progressive group for the game |
| ResetId | The reset ID |
| SASPoll | The code sent the gaming machine to request information |

### CriticalMeterMovement

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |

### SAS_EVENT_CODE_54_PROGRESSIVE_WIN

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### MachineInfo

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Data

| field | description |
|---|-----|
| AssetNumber | The asset number as reported by the Bally system |
| CabinetType | The cabinet type as reported by the Bally system |
| Denom | The denom of the game as reported by the Bally system |
| GameTheme | The game theme as reported by the Bally system |
| HoldPct | The hold percentage for this gaming machine as reported by the Bally system |
| Paylines | The paylines for this gaming machine as reported by the Bally system |
| SealNumber | The seal number of the gaming machine as reported by the Bally system |
| GLIConfirmation | The GLI confirmation number for this gaming machine as reported by the Bally system |
| HostDeviceID | The unique identifier of the gaming machine as reported by the Bally system |
| PurchaseDate | The date on which the casino purchased this gaming machine as reported by the Bally system |
| Reels | The reels for this game as reported by the Bally system |
| Bank | The bank at which this game is located as reported by the Bally system |
| Crc | Used for internal diagnostics by Acres 4 |
| Mfr | The manufacturer name as reported by the Bally system |
| SasSerialNumber | The serial number for this gaming machine as reported by the machine itself |
| Seat | The seat at which this gaming machine is located as reported by the Bally system |
| Section | The section at which this gaming machine is located as reported by the Bally system |
| StartDate | The date on which the casino put this machine into service as reported by the Bally system |
| EndDate | The date on which the casino removed this gaming machine from use as reported by the Bally system |
| LastUpdate | The last time this gaming machine was updated in the Acres 4 system |
| Model | The model for this gaming machine as reported by the Bally system |
| ProgramNumber | The program number for this gaming machine as reported by the Bally system |
| SerialNumber | The serial number for this gaming machine as reported by the gaming machine |
| TokenDenom | The token denomination as reported by the Bally system |

### SAS_EVENT_CODE_18_AC_POWER_WAS_LOST_FROM_GAMING_MACHINE

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| SASPoll | The code sent the gaming machine to request information |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |

### SAS_EVENT_CODE_85_SELF_TEST_OR_OPERATOR_MENU_HAS_BEEN_EXITED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |

### SAS_EVENT_CODE_2F_BILL_ACCEPTOR_VERSION_CHANGED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_11_SLOT_DOOR_WAS_OPENED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SendEnabledFeatures

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CurrentCredits | Value of the current credit meter as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid credits canceled as reported by the gaming machine |
| Jackpot | Total value of all jackpots as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| Reserved | Reserved |
| Features | The featrues that are enabled |
| GameNumber | The game number currently in use |

### SAS_EVENT_CODE_8C_GAME_SELECTED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |
| GameNumber | The number of the game being recalled or selected. Use the SASEvent field to determine the meaning of this key |

### SAS_EVENT_CODE_15_CARD_CAGE_WAS_OPENED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_7F_GAME_HAS_ENDED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SendLastAcceptedBillInformation

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Value of the current credit meter as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid credits canceled as reported by the gaming machine |
| Jackpot | Total value of all jackpots as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| Bills | Indicates the number of bills accepted |
| Country | The country code for the bills accepted |
| Denom | The denom of the game as reported by the Bally system |

### SAS_POLL_54_SEND_SAS_VERSION_ID_AND_GAMING_MACHINE_SERIAL_NUMBER

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| SerialNumber | The serial number for this gaming machine as reported by the gaming machine |
| Length | The length of the packet |
| SasVersionNumber | The SAS version number this gaming machine implements |

### SAS_EVENT_CODE_9A_POWER_OFF_CASHBOX_DOOR_ACCESS

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_1A_CASHBOX_DOOR_WAS_CLOSED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_66_CASH_OUT_BUTTON_PRESSED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_99_POWER_OFF_SLOT_DOOR_ACCESS

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_2B_BILL_REJECTED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_78_PRINTER_CARRIAGE_JAMMED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SendCurrentHopperStatus

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Value of the current credit meter as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid credits canceled as reported by the gaming machine |
| Jackpot | Total value of all jackpots as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| Length | The length of the packet |
| Level | The handpay level as reported by the gaming machine |
| PercentFull | How full the hopper is expressed as a percent |
| Status | The current hopper status |

### SAS_EVENT_CODE_27_CASHBOX_FULL_DETECTED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_86_GAMING_MACHINE_IS_OUT_OF_SERVICE

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_70_EVENT_BUFFER_OVERFLOW

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Data

| field | description |
|---|-----|
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |

### SendSASVersionIDAndGamingMachineSerialNumber

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Value of the current credit meter as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid credits canceled as reported by the gaming machine |
| Jackpot | Total value of all jackpots as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| Length | The length of the packet |
| SasVersionNumber | The SAS version number this gaming machine implements |
| SerialNumber | The serial number for this gaming machine as reported by the gaming machine |

### SendEnabledGameNumbers

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Value of the current credit meter as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid credits canceled as reported by the gaming machine |
| Jackpot | Total value of all jackpots as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| EnabledGameNumbers | The numbers for the games that are enabled |
| Length | The length of the packet |
| NumberOfGames | Total games implemented by this gaming machine |

### SAS_POLL_8F_SEND_PHYSICAL_REEL_STOP_INFORMATION

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| PhysicalReelStops | The current physical reel stops |

### SAS_EVENT_CODE_82_DISPLAY_METERS_OR_ATTENDANT_MENU_HAS_BEEN_ENTERED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_1D_BELLY_DOOR_WAS_OPENED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_43_REEL_3_TILT

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_13_DROP_DOOR_WAS_OPENED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_EVENT_CODE_52_HANDPAY_WAS_RESET

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |

### SendSelectedGameNumber

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| Jackpot | Total value of all jackpots as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Value of the current credit meter as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid credits canceled as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| SelectedGameNumber | Indicates which game is being selected |

### SAS_EVENT_CODE_84_SELF_TEST_OR_OPERATOR_MENU_HAS_BEEN_ENTERED

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_POLL_3D_SEND_CASH_OUT_TICKET_INFORMATION

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| TicketNumber | The ticket number |
| AmountInCents | The value of the cash out ticket in cents |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASPoll | The code sent the gaming machine to request information |

### SAS_POLL_4F_SEND_CURRENT_HOPPER_STATUS

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| Length | The length of the packet |
| Level | The handpay level as reported by the gaming machine |
| PercentFull | How full the hopper is expressed as a percent |
| Status | The current hopper status |

### SAS_EVENT_CODE_20_GENERAL_TILT

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| SASPoll | The code sent the gaming machine to request information |
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |

### SendTotalNumberOfGamesImplemented

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| HandpayCancelCredit | Total value of hand paid credits canceled as reported by the gaming machine |
| Jackpot | Total value of all jackpots as reported by the gaming machine |
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Value of the current credit meter as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| NumberOfGames | Total games implemented by this gaming machine |

### SAS_EVENT_CODE_98_POWER_OFF_CARD_CAGE_ACCESS

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to globally order events in the Foundation event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the Bally system |
| host_id_player | The unique identifier for the player in the Bally system |
| player_card_number | The player's card number |

#### Meters

| field | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of hand paid canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |

#### Data

| field | description |
|---|-----|
| MIBTime | The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |
