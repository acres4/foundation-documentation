# Foundation Event Replay API

## Connection

### Real Time Replay

Users can connect to a real time event replay websocket at `wss://<property>.kailabor.com/nex7/sas-reader/ws/real-time-event-streamer` where `<property>` is the subdomain for the casino running Foundation. Clients enter parameters determining which events to stream as URL parameters following `?`. The API interprets the parameters as follows:

| Field | Description |
| --- | ------- |
| raw-json | Send data to the client as a flattened packet described by the data dictionary below. The value should simply be set to `true` |
| sas_codes | A comma delimited list of real time SAS exception codes. If set, the server will only stream data matching these SAS exception codes to the client. The full list of real time SAS exception codes can be found in Slot Accounting System Protocol Version 6.02 Appendix A Table A-1. Enter codes in decimal notation. |
| asset_number | A comma delimited list of asset numbers. If set, the server will send data matching these asset numbers. Note that if `sas_serial_number` is also set, the server will apply an `OR` filter to the data |
| sas_serial_number | A comma delimited list of SAS serial numbers. If set, the server will send data matching these SAS serial numbers. Note that if `asset_number` is also set, the server will apply an `OR` filter to the data |
| start_idx | Sets the starting `idx` field from which to read data. Acres4 uses this field to apply a global ordering to all data packets in the stream |

While this endpoint is meant for streaming real time events, the server maintains a cache of up to 10,000 events that are up to 10 minutes old, whichever cache is smaller. The `start_idx` filter allows for users to look slightly into the past for older events. An example URL is

`wss://mycasino.kailabor.com/nex7/sas-reader/ws/real-time-event-streamer?raw-json=true&start_idx=161526`

Users must upgrade all requests to a websocket to avoid a `400` error from the server.

The server streams all packets as an array of flat packets, even if there is only once flat packet sent at a time.

### Historical Replay

Users connect to the historical event stream replay API over a websocket located at `wss://<property>.kailabor.com/nex7/sas-reader/ws/historical-event-streamer` where `<property>` is the subdomain for the casino running Foundation. Clients enter parameters determining which events to stream as URL parameters following `?`. The API interprets the parameters as follows:

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

If using `start` and `end` filters, leave `start_idx` and `end_idx` fields blank. Likewise, if using `start_idx` and `end_idx` fields, leave `start` and `end` fields blank. Because indexes are sequential, any `limit` set on a query bounded by `start_idx` and `end_idx` is ignored. Use these fields to bound your query instead. An example URL for reading data is as follows:

`wss://mycasino.kailabor.com/nex7/sas-reader/ws/historical-event-streamer?start=1561705200000000000&end=1562184823000000000&limit=50000&raw-json=true`

Users must upgrade all requests to a websocket to avoid a `400` error from the server.

The server streams all packets as an array of flat packets, even if there is only once flat packet sent at a time.

## Performance

Acres 4 have tested the websocket connection with limits up to 1,000,000 packets. This limit is placed on the number of packets processed from the underlying data structure through the server and may not reflect the number of flattened data packets. Some event packets do not yield a flattened packet and others yield multiple packets. Packets that do not yield a flattened packet are used by Acres 4 for internal purposes only. Processing the same underlying packets yields the same set of flattened packets across multiple trials.

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
    "event_time":1561852793880,
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

### Meter Movement

Starting with version 1.3, Foundation reports meter changes with a new packet type `PACKET_SAS_METERS_CHANGE`. Data in this packet contains key value pairs of `<meter name> : <meter value>` where `<meter name>` is a string representation of any of the meters mentioned in `Slot Accounting System Protocol Version 6.02 Appendix C Table C-7` or a standard Foundation data field. A full list of the meter codes we use is at the bottom of this document. An example packet is below:

```
{
    "type":"PACKET_SAS_METERS_CHANGE",
    "sas_serial_number":"0-2459-46397",
    "player_card_number":"",
    "asset_number":"1234",
    "host_id_player":"",
    "idx":37656,
    "cache_time":1565039283259,
    "data": {
        "CURRENT_CREDITS":19550,
        "ELECTRONIC_REGULAR_CASHABLE_TRANSFERS_TO_GAMING_MACHINE_CREDITS":288000,
        "ELECTRONIC_REGULAR_CASHABLE_TRANSFERS_TO_HOST_CREDITS":103600,
        "GAMES_LOST":544,
        "GAMES_PLAYED":627,
        "GAMES_WON":83,
        "IN_HOUSE_CASHABLE_TRANSFERS_TO_GAMING_MACHINE_CENTS":288000,
        "IN_HOUSE_CASHABLE_TRANSFERS_TO_HOST_CENTS":103600,
        "IN_HOUSE_TRANSFERS_TO_GAMING_MACHINE_THAT_INCLUDED_CASHABLE_AMOUNTS_QUANTITY":27,
        "IN_HOUSE_TRANSFERS_TO_HOST_THAT_INCLUDED_CASHABLE_AMOUNTS_QUANTITY":20,
        "MIBTime":1562083037909,
        "REGULAR_CASHABLE_TICKET_IN_CENTS":19850,
        "REGULAR_CASHABLE_TICKET_IN_QUANTITY":8,
        "REGULAR_CASHABLE_TICKET_OUT_CENTS":159600,
        "REGULAR_CASHABLE_TICKET_OUT_QUANTITY":21,
        "SPECIAL_METERS_LEGACY_BONUS_DEDUCTIBLE_METER":159700,
        "SPECIAL_METERS_POWER_RESET":129,"SPECIAL_METERS_SLOT_DOOR_OPEN":123,
        "SPECIAL_METERS_TOTAL_CREDIT_VALUE_OF_BILLS":100,
        "SPECIAL_METERS_TOTAL_DOLLAR_VALUE_OF_BILLS":1,
        "TOTAL_CANCELLED_CREDITS":429150,
        "TOTAL_CASHABLE_TICKET_IN_CREDITS":19850,
        "TOTAL_CASHABLE_TICKET_OUT_CREDITS":159600,
        "TOTAL_COIN_IN_CREDITS":31225,
        "TOTAL_COIN_OUT_CREDITS":185075,
        "TOTAL_CREDITS_FROM_BILLS_ACCEPTED":100,
        "TOTAL_DROP_CREDITS":307950,
        "TOTAL_ELECTRONIC_TRANSFERS_TO_GAMING_MACHINE_CREDITS":288000,
        "TOTAL_ELECTRONIC_TRANSFERS_TO_HOST_CREDITS":103600,
        "TOTAL_HAND_PAID_CANCELLED_CREDITS":165950,
        "TOTAL_HAND_PAID_CREDITS":165950,
        "TOTAL_MACHINE_PAID_EXTERNAL_BONUS_WIN_CREDITS":159700,
        "TOTAL_MACHINE_PAID_PAYTABLE_WIN_CREDITS":25375,
        "TOTAL_NUMBER_OF_1_BILLS_ACCEPTED":1,
        "TOTAL_REGULAR_CASHABLE_TICKET_IN_CREDITS":19850,
        "TOTAL_SAS_CASHABLE_TICKET_IN_CENTS":19850,
        "TOTAL_SAS_CASHABLE_TICKET_IN_QUANTITY":8,
        "TOTAL_SAS_CASHABLE_TICKET_OUT_CENTS":159600,
        "TOTAL_SAS_CASHABLE_TICKET_OUT_QUANTITY":21,
        "TOTAL_TICKET_IN_CREDITS":19850,
        "TOTAL_TICKET_OUT_CREDITS":159600,
        "TOTAL_WON_CREDITS":185075,
        "meter_snap_flag":129,
        "num_meters":41
    }
}
```

## Data Dictionary

All packets other than `MachineInfo` and `PlayerInfo` contain the following fields

| field | description |
|---|-----|
| type | The type of data carried in this packet. |
| idx | Monotonically increasing uint64 value used to glohost order events in the Nex7 event stream |
| cache_time | Time at which the global event collector received this event, represented as milliseconds since the start of the Unix epoch |
| event_time | The time at which the Machine Interface Board (MIB) received this event |
| sas_serial_number | String identifier for the machine at which a packet originated. |
| asset_number | The asset number for the gaming machine as reported by the host system |
| host_id_player | The unique identifier for the player in the host system |
| player_card_number | The player's card number |

and the following meters

| meter | description |
|---|-----|
| CoinIn | Total value of coin in as reported by the gaming machine |
| CoinOut | Total value of coin out as reported by the gaming machine |
| CurrentCredits | Current credit meter value as reported by the gaming machine |
| GamesPlayed | Total number of games played as reported by the gaming machine |
| GamesWon | Total number of games won as reported by the gaming machine |
| HandpayCancelCredit | Total value of handpay canceled credits as reported by the gaming machine |
| Jackpot | Total value of jackpots paid as reported by the gaming machine |

Lastly, many but not all packets contain the following fields in the `data` portion of the packet:

| field | description |
|---|-----|
| MIBTime | (DEPRECATED -- will be removed soon) The time at which the Machine Interface Board (MIB) received this event |
| SASEvent | The SAS exception code for this real time SAS event |
| SASPoll | The code sent the gaming machine to request information |

Data packets that only contain fields listed above are marked with `Standard fields only` in the documentation below. Other fields that are more specific are listed under the packet type name.

### CriticalMeterMovement

| field | description |
|---|-----|
| HostIDPlayer | The unique identifier for this player as reported by the host system |
| MIBTime | (DEPRECATED -- will be removed soon) The time at which the Machine Interface Board (MIB) received this event |

### MachineInfo

| field | description |
|---|-----|
| Bank | The bank at which this game is located as reported by the host system |
| Crc | Used for internal diagnostics by Acres 4 |
| Denom | The denom of the game as reported by the host system |
| EndDate | The date on which the casino removed this gaming machine from use as reported by the host system |
| HostDeviceID | The unique identifier of the gaming machine as reported by the host system |
| Paylines | The paylines for this gaming machine as reported by the host system |
| PurchaseDate | The date on which the casino purchased this gaming machine as reported by the host system |
| SasSerialNumber | The serial number for this gaming machine as reported by the machine itself |
| Seat | The seat at which this gaming machine is located as reported by the host system |
| AssetNumber | The asset number as reported by the host system |
| CabinetType | The cabinet type as reported by the host system |
| LastUpdate | The last time this gaming machine was updated in the Acres 4 system |
| Model | The model for this gaming machine as reported by the host system |
| ProgramNumber | The program number for this gaming machine as reported by the host system |
| Reels | The reels for this game as reported by the host system |
| SealNumber | The seal number of the gaming machine as reported by the host system |
| TokenDenom | The token denomination as reported by the host system |
| GLIConfirmation | The GLI confirmation number for this gaming machine as reported by the host system |
| GameTheme | The game theme as reported by the host system |
| HoldPct | The hold percentage for this gaming machine as reported by the host system |
| Mfr | The manufacturer name as reported by the host system |
| Section | The section at which this gaming machine is located as reported by the host system |
| SerialNumber | The serial number for this gaming machine as reported by the gaming machine |
| StartDate | The date on which the casino put this machine into service as reported by the host system |

### PlayerInfo

| field | description |
|---|-----|
| LastEventTime | The last time at which Acres 4 saw an event for this player |
| LastLocation | The last location of the player |
| LastName | The last name of this player |
| LoyaltyTierStr | Same as MembershipTierStr |
| Banned | If true, the player is banned from this casino |
| BirthDate | The date of birth for this player |
| CacheTime | The time at which the Acres 4 global event collector collected this event |
| CardNumber | The card number for this player |
| ZipCode | The zip code for this player |
| ZipExt | The zip code extension for the player |
| HostClubNumber | The player number as reported by the host system |
| MembershipTierStr | The string representation of the membership tier level for this player |
| Theo | The theoretical win for this player |
| Crc | Used for internal diagnostics by Acres 4 |
| FirstName | The first name of the player |
| Gender | The gender for this player |
| HostIdPlayer | The unique identifier for this player as reported by the host system |

### SAS_EVENT_CODE_11_SLOT_DOOR_WAS_OPENED

Standard fields only

### SAS_EVENT_CODE_12_SLOT_DOOR_WAS_CLOSED

Standard fields only

### SAS_EVENT_CODE_13_DROP_DOOR_WAS_OPENED

Standard fields only

### SAS_EVENT_CODE_14_DROP_DOOR_WAS_CLOSED

Standard fields only

### SAS_EVENT_CODE_15_CARD_CAGE_WAS_OPENED

Standard fields only

### SAS_EVENT_CODE_16_CARD_CAGE_WAS_CLOSED

Standard fields only

### SAS_EVENT_CODE_17_AC_POWER_WAS_APPLIED_TO_GAMING_MACHINE

Standard fields only

### SAS_EVENT_CODE_18_AC_POWER_WAS_LOST_FROM_GAMING_MACHINE

Standard fields only

### SAS_EVENT_CODE_19_CASHBOX_DOOR_WAS_OPENED

Standard fields only

### SAS_EVENT_CODE_1A_CASHBOX_DOOR_WAS_CLOSED

Standard fields only

### SAS_EVENT_CODE_1B_CASHBOX_WAS_REMOVED

Standard fields only

### SAS_EVENT_CODE_1C_CASHBOX_WAS_INSTALLED

Standard fields only

### SAS_EVENT_CODE_1D_BELLY_DOOR_WAS_OPENED

Standard fields only

### SAS_EVENT_CODE_1E_BELLY_DOOR_WAS_CLOSED

Standard fields only

### SAS_EVENT_CODE_20_GENERAL_TILT

Standard fields only

### SAS_EVENT_CODE_27_CASHBOX_FULL_DETECTED

Standard fields only

### SAS_EVENT_CODE_28_BILL_JAM

| field | description |
|---|-----|
| HostIDPlayer | The unique identifier for this player as reported by the host system |

### SAS_EVENT_CODE_29_BILL_ACCEPTOR_HARDWARE_FAILURE

| field | description |
|---|-----|
| HostIDPlayer | The unique identifier for this player as reported by the host system |

### SAS_EVENT_CODE_2B_BILL_REJECTED

| field | description |
|---|-----|
| HostIDPlayer | The unique identifier for this player as reported by the host system |

### SAS_EVENT_CODE_2F_BILL_ACCEPTOR_VERSION_CHANGED

Standard fields only

### SAS_EVENT_CODE_32_CMOS_RAM_ERROR

Standard fields only

### SAS_EVENT_CODE_34_EEPROM_ERROR

Standard fields only

### SAS_EVENT_CODE_3C_OPERATOR_CHANGED_OPTIONS

Standard fields only

### SAS_EVENT_CODE_3D_A_CASH_OUT_TICKET_HAS_BEEN_PRINTED

| field | description |
|---|-----|
| HostIDPlayer | The unique identifier for this player as reported by the host system |

### SAS_EVENT_CODE_42_REEL_2_TILT

Standard fields only

### SAS_EVENT_CODE_43_REEL_3_TILT

Standard fields only

### SAS_EVENT_CODE_4F_BILL_ACCEPTED

| field | description |
|---|-----|
| Bills | Indicates the number of bills accepted |
| CountryCode | Indicates the country code for the bill accepted |
| DenomCode | Indicates the denomination code for the bill accepted |
| HostIDPlayer | The unique identifier for this player as reported by the host system |

### SAS_EVENT_CODE_51_HANDPAY_IS_PENDING

Standard fields only

### SAS_EVENT_CODE_52_HANDPAY_WAS_RESET

Standard fields only

### SAS_EVENT_CODE_53_NO_PROGRESSIVE_INFORMATION_HAS_BEEN_RECEIVED_FOR_5_SECONDS

Standard fields only

### SAS_EVENT_CODE_54_PROGRESSIVE_WIN

| field | description |
|---|-----|
| HostIDPlayer | The unique identifier for this player as reported by the host system |

### SAS_EVENT_CODE_60_PRINTER_COMMUNICATION_ERROR

Standard fields only

### SAS_EVENT_CODE_61_PRINTER_PAPER_OUT_ERROR

| field | description |
|---|-----|
| HostIDPlayer | The unique identifier for this player as reported by the host system |

### SAS_EVENT_CODE_66_CASH_OUT_BUTTON_PRESSED

Standard fields only

### SAS_EVENT_CODE_70_EVENT_BUFFER_OVERFLOW

Standard fields only

### SAS_EVENT_CODE_71_CHANGE_LAMP_ON

Standard fields only

### SAS_EVENT_CODE_72_CHANGE_LAMP_OFF

Standard fields only

### SAS_EVENT_CODE_74_PRINTER_PAPER_LOW

| field | description |
|---|-----|
| HostIDPlayer | The unique identifier for this player as reported by the host system |

### SAS_EVENT_CODE_78_PRINTER_CARRIAGE_JAMMED

| field | description |
|---|-----|
| HostIDPlayer | The unique identifier for this player as reported by the host system |

### SAS_EVENT_CODE_7B_BILL_VALIDATOR

Standard fields only

### SAS_EVENT_CODE_7E_GAME_HAS_STARTED

| field | description |
|---|-----|
| ProgressiveGroup | The progressive group as reported by the gaming machine |
| TotalCoinInMeter | The value of the total coin in meter at the start of the game |
| WagerType | The wager type |
| CreditsWagered | The number of credits wagered on this game |
| HostIDPlayer | The unique identifier for this player as reported by the host system |

### SAS_EVENT_CODE_7F_GAME_HAS_ENDED

| field | description |
|---|-----|
| GameWin | The amount won from the game |
| HostIDPlayer | The unique identifier for this player as reported by the host system |

### SAS_EVENT_CODE_82_DISPLAY_METERS_OR_ATTENDANT_MENU_HAS_BEEN_ENTERED

Standard fields only

### SAS_EVENT_CODE_83_DISPLAY_METERS_OR_ATTENDANT_MENU_HAS_BEEN_EXITED

Standard fields only

### SAS_EVENT_CODE_84_SELF_TEST_OR_OPERATOR_MENU_HAS_BEEN_ENTERED

Standard fields only

### SAS_EVENT_CODE_85_SELF_TEST_OR_OPERATOR_MENU_HAS_BEEN_EXITED

Standard fields only

### SAS_EVENT_CODE_86_GAMING_MACHINE_IS_OUT_OF_SERVICE

Standard fields only

### SAS_EVENT_CODE_88_REEL_N_HAS_STOPPED

| field | description |
|---|-----|
| PhysicalStop | The position in which the reel stopped |
| ReelNum | The number of the reel for which a stop is being reported |

### SAS_EVENT_CODE_89_COIN_CREDIT_WAGERED

Standard fields only

### SAS_EVENT_CODE_8A_GAME_RECALL_ENTRY_HAS_BEEN_DISPLAYED

| field | description |
|---|-----|
| GameNumber | The number of the game being selected. |
| HostIDPlayer | The unique identifier for this player as reported by the host system |
| RecallIndex | The recall index for the game being recalled |

### SAS_EVENT_CODE_8C_GAME_SELECTED

| field | description |
|---|-----|
| GameNumber | The number of the game being selected. |

### SAS_EVENT_CODE_98_POWER_OFF_CARD_CAGE_ACCESS

Standard fields only

### SAS_EVENT_CODE_99_POWER_OFF_SLOT_DOOR_ACCESS

Standard fields only

### SAS_EVENT_CODE_9A_POWER_OFF_CASHBOX_DOOR_ACCESS

Standard fields only

### SAS_EVENT_CODE_9B_POWER_OFF_DROP_DOOR_ACCESS

Standard fields only

### SAS_POLL_1B_SEND_HANDPAY_INFORMATION

| field | description |
|---|-----|
| PartialPayAmount | The partial pay amount |
| ProgressiveGroup | The progressive group as reported by the gaming machine |
| ResetId | The reset ID |
| Unused | N/A |
| Amount | The amount to be paid by this handpay |
| Level | The hopper level as reported by the gaming machine |

### SAS_POLL_1F_SEND_GAMING_MACHINE_ID_AND_INFORMATION

| field | description |
|---|-----|
| PaytableId | The paytable ID |
| ProgressiveGroup | The progressive group as reported by the gaming machine |
| AdditionalId | Additional ID information |
| BasePercentage | The base percentage |
| Denomination | The denomination |
| GameId | The game ID |
| GameOptions | The game options |
| MaxBet | The maximum bet |

### SAS_POLL_3D_SEND_CASH_OUT_TICKET_INFORMATION

| field | description |
|---|-----|
| AmountInCents | The value of the cash out ticket in cents |
| HostIDPlayer | The unique identifier for this player as reported by the host system |
| TicketNumber | The ticket number |

### SAS_POLL_48_SEND_LAST_ACCEPTED_BILL_INFORMATION

| field | description |
|---|-----|
| Bills | Indicates the number of bills accepted |
| Country | The country code for the bills accepted |
| Denom | The denom of the game as reported by the host system |

### SAS_POLL_4F_SEND_CURRENT_HOPPER_STATUS

| field | description |
|---|-----|
| Length | Internal use only |
| Level | The hopper level as reported by the gaming machine |
| PercentFull | How full the hopper is expressed as a percent |
| Status | The current hopper status |

### SAS_POLL_51_SEND_TOTAL_NUMBER_OF_GAMES_IMPLEMENTED

| field | description |
|---|-----|
| NumberOfGames | Total games implemented by this gaming machine |

### SAS_POLL_54_SEND_SAS_VERSION_ID_AND_GAMING_MACHINE_SERIAL_NUMBER

| field | description |
|---|-----|
| SasVersionNumber | The SAS version number this gaming machine implements |
| SerialNumber | The serial number for this gaming machine as reported by the gaming machine |
| Length | Internal use only |

### SAS_POLL_55_SEND_SELECTED_GAME_NUMBER

| field | description |
|---|-----|
| SelectedGameNumber | Indicates which game is being selected |

### SAS_POLL_56_SAS_SEND_ENABLED_GAME_NUMBERS

| field | description |
|---|-----|
| EnabledGameNumbers | The numbers for the games that are enabled |
| Length | Internal use only |
| NumberOfGames | Total games implemented by this gaming machine |

### SAS_POLL_8F_SEND_PHYSICAL_REEL_STOP_INFORMATION

| field | description |
|---|-----|
| PhysicalReelStops | The current physical reel stops |

### SAS_POLL_A0_SEND_ENABLED_FEATURES

| field | description |
|---|-----|
| Features | The features that are enabled |
| GameNumber | The number of the game being selected. |
| Reserved | Reserved |

### SAS_POLL_A8_ENABLE_JACKPOT_HANDPAY_RESET_METHOD

| field | description |
|---|-----|
| AckCode | Used by Acres 4 to implement the SAS protocol |

### SAS_POLL_FF_REAL_TIME_EXCEPTION

| field | description |
|---|-----|
| HostIDPlayer | The unique identifier for this player as reported by the host system |

### SendCurrentHopperStatus

| field | description |
|---|-----|
| Length | Internal use only |
| Level | The hopper level as reported by the gaming machine |
| PercentFull | How full the hopper is expressed as a percent |
| Status | The current hopper status |

### SendEnabledFeatures

| field | description |
|---|-----|
| GameNumber | The game number currently in use |
| Reserved | Reserved |
| Features | The features that are enabled |

### SendEnabledGameNumbers

| field | description |
|---|-----|
| EnabledGameNumbers | The numbers for the games that are enabled |
| Length | Internal use only |
| NumberOfGames | Total games implemented by this gaming machine |

### SendGamingMachineIDAndInformation

| field | description |
|---|-----|
| PaytableId | The paytable ID |
| ProgressiveGroup | The progressive group as reported by the gaming machine |
| AdditionalId | Additional ID information |
| BasePercentage | The base percentage |
| Denomination | The denomination |
| GameId | The game ID |
| GameOptions | The game options |
| MaxBet | The maximum bet |

### SendLastAcceptedBillInformation

| field | description |
|---|-----|
| Denom | The denom of the bills accepted |
| Bills | Indicates the number of bills accepted |
| Country | The country code for the bills accepted |

### SendPhysicalReelStopInformation

| field | description |
|---|-----|
| PhysicalReelStops | The current physical reel stops |

### SendSASVersionIDAndGamingMachineSerialNumber

| field | description |
|---|-----|
| Length | Internal use only |
| SasVersionNumber | The SAS version number this gaming machine implements |
| SerialNumber | The serial number for this gaming machine as reported by the gaming machine |

### SendSelectedGameNumber

| field | description |
|---|-----|
| SelectedGameNumber | Indicates which game is being selected |

### SendTotalNumberOfGamesImplemented

| field | description |
|---|-----|
| NumberOfGames | Total games implemented by this gaming machine |

## List of Meter Names Used by Foundation

BONUS_CASHABLE_TRANSFERS_TO_GAMING_MACHINE_CENTS
BONUS_NONRESTRICTED_TRANSFERS_TO_GAMING_MACHINE_CENTS
BONUS_TRANSFERS_TO_GAMING_MACHINE_THAT_INCLUDED_CASHABLE_AMOUNTS_QUANTITY
BONUS_TRANSFERS_TO_GAMING_MACHINE_THAT_INCLUDED_NONRESTRICTED_AMOUNTS_QUANTITY
CURRENT_CREDITS
CURRENT_RESTRICTED_CREDITS
DEBIT_TICKET_OUT_CENTS
DEBIT_TICKET_OUT_QUANTITY
DEBIT_TRANSFERS_TO_GAMING_MACHINE_CENTS
DEBIT_TRANSFERS_TO_GAMING_MACHINE_QUANTITY
DEBIT_TRANSFERS_TO_TICKET_CENTS
DEBIT_TRANSFERS_TO_TICKET_QUANTITY
ELECTRONIC_DEBIT_TRANSFERS_TO_GAMING_MACHINE_CREDITS
ELECTRONIC_NONRESTRICTED_PROMOTIONAL_TRANSFERS_TO_GAMING_MACHINE_CREDITS
ELECTRONIC_NONRESTRICTED_PROMOTIONAL_TRANSFERS_TO_HOST_CREDITS
ELECTRONIC_REGULAR_CASHABLE_TRANSFERS_TO_GAMING_MACHINE_CREDITS
ELECTRONIC_REGULAR_CASHABLE_TRANSFERS_TO_HOST_CREDITS
ELECTRONIC_RESTRICTED_PROMOTIONAL_TRANSFERS_TO_GAMING_MACHINE_CREDITS
ELECTRONIC_RESTRICTED_PROMOTIONAL_TRANSFERS_TO_HOST_CREDITS
GAMES_LOST
GAMES_PLAYED
GAMES_SINCE_LAST_POWER_RESET
GAMES_SINCE_SLOT_DOOR_CLOSURE
GAMES_WON
IN_HOUSE_CASHABLE_TRANSFERS_TO_GAMING_MACHINE_CENTS
IN_HOUSE_CASHABLE_TRANSFERS_TO_HOST_CENTS
IN_HOUSE_CASHABLE_TRANSFERS_TO_TICKET_CENTS
IN_HOUSE_CASHABLE_TRANSFERS_TO_TICKET_QUANTITY
IN_HOUSE_NONRESTRICTED_TRANSFERS_TO_GAMING_MACHINE_CENTS
IN_HOUSE_NONRESTRICTED_TRANSFERS_TO_HOST_CENTS
IN_HOUSE_RESTRICTED_TRANSFERS_TO_GAMING_MACHINE_CENTS
IN_HOUSE_RESTRICTED_TRANSFERS_TO_HOST_CENTS
IN_HOUSE_RESTRICTED_TRANSFERS_TO_TICKET_CENTS
IN_HOUSE_RESTRICTED_TRANSFERS_TO_TICKET_QUANTITY
IN_HOUSE_TRANSFERS_TO_GAMING_MACHINE_THAT_INCLUDED_CASHABLE_AMOUNTS_QUANTITY
IN_HOUSE_TRANSFERS_TO_GAMING_MACHINE_THAT_INCLUDED_NONRESTRICTED_AMOUNTS_QUANTITY
IN_HOUSE_TRANSFERS_TO_GAMING_MACHINE_THAT_INCLUDED_RESTRICTED_AMOUNTS_QUANTITY
IN_HOUSE_TRANSFERS_TO_HOST_THAT_INCLUDED_CASHABLE_AMOUNTS_QUANTITY
IN_HOUSE_TRANSFERS_TO_HOST_THAT_INCLUDED_NONRESTRICTED_AMOUNTS_QUANTITY
IN_HOUSE_TRANSFERS_TO_HOST_THAT_INCLUDED_RESTRICTED_AMOUNTS_QUANTITY
NONRESTRICTED_TICKET_IN_CENTS
NONRESTRICTED_TICKET_IN_QUANTITY
NUMBER_OF_BILLS_CURRENTLY_IN_THE_STACKER
REGULAR_CASHABLE_TICKET_IN_CENTS
REGULAR_CASHABLE_TICKET_IN_QUANTITY
REGULAR_CASHABLE_TICKET_OUT_CENTS
REGULAR_CASHABLE_TICKET_OUT_QUANTITY
RESERVED_FOR_FUTURE_USE_99
RESERVED_FOR_FUTURE_USE_9A
RESERVED_FOR_FUTURE_USE_9B
RESERVED_FOR_FUTURE_USE_9C
RESERVED_FOR_FUTURE_USE_9D
RESERVED_FOR_FUTURE_USE_9E
RESERVED_FOR_FUTURE_USE_9F
RESERVED_FOR_FUTURE_USE_B2
RESERVED_FOR_FUTURE_USE_B3
RESERVED_FOR_FUTURE_USE_B4
RESERVED_FOR_FUTURE_USE_B5
RESERVED_FOR_FUTURE_USE_B6
RESERVED_FOR_FUTURE_USE_B7
RESTRICTED_TICKET_IN_CENTS
RESTRICTED_TICKET_IN_QUANTITY
RESTRICTED_TICKET_OUT_CENTS
RESTRICTED_TICKET_OUT_QUANTITY
SESSIONS_PLAYED
SPECIAL_METERS_CURRENT_HOPPER_LEVEL
SPECIAL_METERS_LEGACY_BONUS_DEDUCTIBLE_METER
SPECIAL_METERS_LEGACY_BONUS_NON_DEDUCTIBLE_METER
SPECIAL_METERS_LEGACY_BONUS_WAGER_MATCH
SPECIAL_METERS_POWER_RESET
SPECIAL_METERS_SLOT_DOOR_OPEN
SPECIAL_METERS_TOTAL_CREDIT_VALUE_OF_BILLS
SPECIAL_METERS_TOTAL_DOLLAR_VALUE_OF_BILLS
SPECIAL_METERS_TOURNAMENT_CREDIT_WAGERED
SPECIAL_METERS_TOURNAMENT_CREDIT_WON
SPECIAL_METERS_TOURNAMENT_GAMES_PLAYED
SPECIAL_METERS_TOURNAMENT_GAMES_WON
SPECIAL_METERS_TRUE_COIN_IN
SPECIAL_METERS_TRUE_COIN_OUT
TOTAL_ATTENDANT_PAID_EXTERNAL_BONUS_WIN_CREDITS
TOTAL_ATTENDANT_PAID_PAYTABLE_WIN_CREDITS
TOTAL_ATTENDANT_PAID_PROGRESSIVE_WIN_CREDITS
TOTAL_CANCELLED_CREDITS
TOTAL_CASHABLE_TICKET_IN_CREDITS
TOTAL_CASHABLE_TICKET_OUT_CREDITS
TOTAL_CASHABLE_TICKET_OUT_QUANTITY
TOTAL_COIN_IN_CREDITS
TOTAL_COIN_OUT_CREDITS
TOTAL_CREDITS_FROM_BILLS_ACCEPTED
TOTAL_CREDITS_FROM_BILLS_DISPENSED_FROM_HOPPER
TOTAL_CREDITS_FROM_BILLS_DIVERTED_TO_HOPPER
TOTAL_CREDITS_FROM_BILLS_TO_DROP
TOTAL_CREDITS_FROM_COINS_TO_DROP
TOTAL_CREDITS_FROM_COIN_ACCEPTOR
TOTAL_CREDITS_FROM_EXTERNAL_COIN_ACCEPTOR
TOTAL_CREDITS_PAID_FROM_HOPPER
TOTAL_DROP_CREDITS
TOTAL_ELECTRONIC_TRANSFERS_TO_GAMING_MACHINE_CREDITS
TOTAL_ELECTRONIC_TRANSFERS_TO_HOST_CREDITS
TOTAL_HAND_PAID_CANCELLED_CREDITS
TOTAL_HAND_PAID_CREDITS
TOTAL_JACKPOT_CREDITS
TOTAL_MACHINE_PAID_EXTERNAL_BONUS_WIN_CREDITS
TOTAL_MACHINE_PAID_PAYTABLE_WIN_CREDITS
TOTAL_MACHINE_PAID_PROGRESSIVE_WIN_CREDITS
TOTAL_NONRESTRICTED_AMOUNT_PLAYED_CREDITS
TOTAL_NONRESTRICTED_PROMOTIONAL_TICKET_IN_CREDITS
TOTAL_NONRESTRICTED_PROMOTIONAL_TICKET_IN_QUANTITY
TOTAL_NUMBER_OF_1000000_BILLS_ACCEPTED
TOTAL_NUMBER_OF_100000_BILLS_ACCEPTED
TOTAL_NUMBER_OF_10000_BILLS_ACCEPTED
TOTAL_NUMBER_OF_1000_BILLS_ACCEPTED
TOTAL_NUMBER_OF_1000_BILLS_DISPENSED_FROM_HOPPER
TOTAL_NUMBER_OF_1000_BILLS_DIVERTED_TO_HOPPER
TOTAL_NUMBER_OF_1000_BILLS_TO_DROP
TOTAL_NUMBER_OF_100_BILLS_ACCEPTED
TOTAL_NUMBER_OF_100_BILLS_DISPENSED_FROM_HOPPER
TOTAL_NUMBER_OF_100_BILLS_DIVERTED_TO_HOPPER
TOTAL_NUMBER_OF_100_BILLS_TO_DROP
TOTAL_NUMBER_OF_10_BILLS_ACCEPTED
TOTAL_NUMBER_OF_10_BILLS_DISPENSED_FROM_HOPPER
TOTAL_NUMBER_OF_10_BILLS_DIVERTED_TO_HOPPER
TOTAL_NUMBER_OF_10_BILLS_TO_DROP
TOTAL_NUMBER_OF_1_BILLS_ACCEPTED
TOTAL_NUMBER_OF_1_BILLS_DISPENSED_FROM_HOPPER
TOTAL_NUMBER_OF_1_BILLS_DIVERTED_TO_HOPPER
TOTAL_NUMBER_OF_1_BILLS_TO_DROP
TOTAL_NUMBER_OF_200000_BILLS_ACCEPTED
TOTAL_NUMBER_OF_20000_BILLS_ACCEPTED
TOTAL_NUMBER_OF_2000_BILLS_ACCEPTED
TOTAL_NUMBER_OF_200_BILLS_ACCEPTED
TOTAL_NUMBER_OF_200_BILLS_DISPENSED_FROM_HOPPER
TOTAL_NUMBER_OF_200_BILLS_DIVERTED_TO_HOPPER
TOTAL_NUMBER_OF_200_BILLS_TO_DROP
TOTAL_NUMBER_OF_20_BILLS_ACCEPTED
TOTAL_NUMBER_OF_20_BILLS_DISPENSED_FROM_HOPPER
TOTAL_NUMBER_OF_20_BILLS_DIVERTED_TO_HOPPER
TOTAL_NUMBER_OF_20_BILLS_TO_DROP
TOTAL_NUMBER_OF_250000_BILLS_ACCEPTED
TOTAL_NUMBER_OF_25000_BILLS_ACCEPTED
TOTAL_NUMBER_OF_2500_BILLS_ACCEPTED
TOTAL_NUMBER_OF_250_BILLS_ACCEPTED
TOTAL_NUMBER_OF_25_BILLS_ACCEPTED
TOTAL_NUMBER_OF_2_BILLS_ACCEPTED
TOTAL_NUMBER_OF_2_BILLS_DISPENSED_FROM_HOPPER
TOTAL_NUMBER_OF_2_BILLS_DIVERTED_TO_HOPPER
TOTAL_NUMBER_OF_2_BILLS_TO_DROP
TOTAL_NUMBER_OF_500000_BILLS_ACCEPTED
TOTAL_NUMBER_OF_50000_BILLS_ACCEPTED
TOTAL_NUMBER_OF_5000_BILLS_ACCEPTED
TOTAL_NUMBER_OF_500_BILLS_ACCEPTED
TOTAL_NUMBER_OF_500_BILLS_DISPENSED_FROM_HOPPER
TOTAL_NUMBER_OF_500_BILLS_DIVERTED_TO_HOPPER
TOTAL_NUMBER_OF_500_BILLS_TO_DROP
TOTAL_NUMBER_OF_50_BILLS_ACCEPTED
TOTAL_NUMBER_OF_50_BILLS_DISPENSED_FROM_HOPPER
TOTAL_NUMBER_OF_50_BILLS_DIVERTED_TO_HOPPER
TOTAL_NUMBER_OF_50_BILLS_TO_DROP
TOTAL_NUMBER_OF_5_BILLS_ACCEPTED
TOTAL_NUMBER_OF_5_BILLS_DISPENSED_FROM_HOPPER
TOTAL_NUMBER_OF_5_BILLS_DIVERTED_TO_HOPPER
TOTAL_NUMBER_OF_5_BILLS_TO_DROP
TOTAL_REGULAR_CASHABLE_TICKET_IN_CREDITS
TOTAL_REGULAR_CASHABLE_TICKET_IN_QUANTITY
TOTAL_RESTRICTED_AMOUNT_PLAYED_CREDITS
TOTAL_RESTRICTED_PROMOTIONAL_TICKET_IN_CREDITS
TOTAL_RESTRICTED_PROMOTIONAL_TICKET_IN_QUANTITY
TOTAL_RESTRICTED_PROMOTIONAL_TICKET_OUT_CREDITS
TOTAL_RESTRICTED_PROMOTIONAL_TICKET_OUT_QUANTITY
TOTAL_SAS_CASHABLE_TICKET_IN_CENTS
TOTAL_SAS_CASHABLE_TICKET_IN_QUANTITY
TOTAL_SAS_CASHABLE_TICKET_OUT_CENTS
TOTAL_SAS_CASHABLE_TICKET_OUT_QUANTITY
TOTAL_SAS_RESTRICTED_TICKET_IN_CENTS
TOTAL_SAS_RESTRICTED_TICKET_IN_QUANTITY
TOTAL_SAS_RESTRICTED_TICKET_OUT_CENTS
TOTAL_SAS_RESTRICTED_TICKET_OUT_QUANTITY
TOTAL_TICKET_IN_CREDITS
TOTAL_TICKET_OUT_CREDITS
TOTAL_VALUE_OF_BILLS_CURRENTLY_IN_THE_STACKER_CREDITS
TOTAL_WON_CREDITS
VALIDATED_NO_RECEIPT_CANCELLED_CREDIT_HANDPAY_CENTS
VALIDATED_NO_RECEIPT_CANCELLED_CREDIT_HANDPAY_QUANTITY
VALIDATED_NO_RECEIPT_JACKPOT_HANDPAY_CENTS
VALIDATED_NO_RECEIPT_JACKPOT_HANDPAY_QUANTITY
VALIDATED_RECEIPT_CANCELLED_CREDIT_HANDPAY_CENTS
VALIDATED_RECEIPT_CANCELLED_CREDIT_HANDPAY_QUANTITY
VALIDATED_RECEIPT_JACKPOT_HANDPAY_CENTS
VALIDATED_RECEIPT_JACKPOT_HANDPAY_QUANTITY
WEIGHTED_AVERAGE_THEORETICAL_PAYBACK_PERCENTAGE_IN_HUNDREDTHS_OF_A_PERCENT
