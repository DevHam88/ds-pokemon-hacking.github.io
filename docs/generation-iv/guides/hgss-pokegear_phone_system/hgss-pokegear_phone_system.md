---
title: Pokegear Phone System
tags:
  - Guide (HeartGold)
  - Guide (SoulSilver)
---

# Pokegear Phone System

> Author(s): [MrHam88](https://github.com/DevHam88/).  
> Research: [pret/pokeheartgold](https://github.com/pret/pokeheartgold), Eclipse (Luna), Senate, BlackShark.  

This guide explains the Pokegear phone system in Pokemon HeartGold and SoulSilver: registering contacts, ordinary outgoing calls, random incoming calls, story/event calls, text archives, trainer rematches, Gym Leader rematches, and post-call gifts.  

The system is spread across several files. The most important distinction is that a **contact**, a **phone conversation**, and a **field script** are different things. A contact is a row in `pmtel_book.dat`; it can point to a message archive and a call handler, but story calls and item collection often use normal field scripts as well.

---

## Overview

HGSS's Pokegear phone system is a set of connected but separate systems. `pmtel_book.dat` is a fixed contact list that defines each contact''s ID, handler type, trainer battle ID, home map, schedule, sorting data, and call settings; save data records which contacts the player has registered plus pending rematches, gifts, and queued calls. When the player calls a number, or when the game decides to ring, the Pokegear code chooses a contact-specific handler: ordinary trainers use compact Overlay 101 header and definition tables to select a message and possibly queue a rematch or gift, while Mom, Elm, Oak, Gym Leaders, Baoba, and other special contacts use dedicated logic. The text itself comes from that contact''s message archive, whose message `0` supplies the contact name. Incoming calls share a timed overworld check: it handles queued system events first, then may choose an eligible registered caller at random; map-header permissions can block calls. Ordinary trainer rematches use the contact''s trainer ID to look up a fixed six-entry battle-ID row in Overlay 26, while field scripts can register contacts or immediately configure and start story calls such as Elm''s stolen-Pokemon call.

The following tables summarise the main data and call paths:

| Type | System | Location | What it stores |
|---|---|---|---|
| Data | Pokegear contact data | `tel/pmtel_book.dat` | Contact type, base trainer ID, map location, item, local-call script, schedule, and random-call bucket. |
| Data | Phone conversation texts | `sPhoneMessageGmm` | The message archive used for that contact's name and phone text. |
| Data | Rematch trainer ID lookup | Overlay 26, file offset `0x20C` | The ordinary trainer rematch battle IDs. |
| Game code | Phone call code | ARM9 and Overlays 2, 26, and 101 | The logic that chooses dialogue for the current contact, direction, map, day, time, and story state. |
| Save file | Saved Pokegear phonebook | N/A | Whether the player has registered the contact. |
| Save file | Persistent phone state | N/A | Pending rematches, pending gifts, and queued event calls. |


The game has three call paths:

| Call path | Direction | Example |
|---|---|---|
| Player selects a number | Outgoing | Calling Elm from the Pokegear menu option. |
| The ring manager chooses a registered contact | Incoming | Youngster Joey calling with ordinary chatter, a rematch, or an item. |
| A field script or system event starts/queues a call | Incoming/event | Elm's stolen-Pokemon call, the Togepi Egg call, Mom's purchase call, Bill's full-PC call. |

---

## Phonebook data

The ROM file is a direct NitroFS file: `tel/pmtel_book.dat`

It is **not** a NARC member. For binary hacking, extract and edit this file directly.

### File layout

The first four bytes are a little-endian entry count (75). They are followed by 20-byte contact records for each entry.

| Offset | Size (bytes) | Decomp field | Meaning |
|:---:|:---:|---|---|
| `00` | 1 | `id` | Contact ID (`PHONE_CONTACT_*`). |
| `01` | 1 | `type` | Phone call handler type. Values 0-15 select: generic, Mom, Elm, Gym Leader, and other specialised call logic; see [Contact type values](#contact-type-values). |
| `02` | 1 | `unk2` | Unknown. No direct reader was found in the current public decomp, but that is not proof it is unused in vanilla binary code; leave it unchanged. |
| `03` | 1 | `trainerClass` | Title label shown in the Pokegear contact list. Standard trainer classes and special phone-contact labels use different text paths; see [Contact-list class text](#contact-list-class-text). |
| `04` | 2 | `trainerId` | The contact's original trainer battle ID, or `TRAINER_NONE` (`0`) for a non-trainer. Ordinary trainer rematches use this to find an overlay 26 rematch lookup row. |
| `06` | 2 | `mapId` | The contact's **map-header ID**. It controls local-call behaviour, random-call filtering, location text, and the limited rematch/gift map marker; see [What `mapId` controls](#what-mapid-controls). |
| `08` | 2 | `gift` | Static item used by the trainer-gift field-script path. |
| `0A` | 2 | `phoneScriptIfLocal` | Generic contact's outgoing phone script when the player is on this contact's `mapId`. |
| `0C` | 1 | `unkC` | Greeting-set index. Values `0`-`7` select a row from the shared phone-greeting table; `FF` suppresses the greeting. |
| `0D` | 1 | `rematchWeekday` | Weekday used by the ordinary phone-trainer schedule and Gym Leader schedule. Other rematch availability conditions also apply; see [Rematch availability conditions](#rematch-availability-conditions). |
| `0E` | 1 | `rematchTimeOfDay` | Morning `0`, day `1`, or night `2`, used with the weekday by those schedule checks. |
| `0F` | 1 | `unkF` | Random incoming-call "probability bucket". The game chooses bucket `0` (50% of the time), bucket `1` (30% of the time), and bucket `2` (20% of the time). |
| `10` | 1 | `sortParam[0]` | Stored Pokegear title-sort rank. |
| `11` | 1 | `sortParam[1]` | Stored Pokegear alphabetical-sort rank. |
| `12` | 1 | `sortParam[2]` | Stored Pokegear location-sort rank. |
| `13` | 1 | `sortParam[3]` | Padding (`00` in vanilla). |

:::caution  
The first four bytes control how many 20-byte records the loader reads and allocates, however, they are **not** a safe "add a new contact" control by themselves.

The game has a hardcoded `NUM_PHONE_CONTACTS = 75` limit. Contact IDs index fixed-size save arrays for registered numbers and `PhoneRematch` state, the phone UI slot data, the contact-to-message-archive table, generic-header blocks, and other tables. Several call paths reject IDs `>= 75` before indexing the phonebook.

Appending a record and increasing the count can make the file loader allocate and read it: it does not make the record usable as a new phone contact. A real contact expansion requires coordinated binary patches and expansions of every contact-ID-indexed table.  
:::

### Contact type values

`type` is a numeric call-handler selector, not a trainer class and not a random-call frequency. The vanilla values are:

| Value | Decomp name | Contacts / effect |
|:---:|---|---|
| `0` | `GENERIC` | Ordinary phone trainers. Uses generic header/script tables. |
| `1` | `MOM` | Mom's savings and gift-queue handler. |
| `2` | `PROF_ELM` | Elm's story-flag, badge, Egg, and Pokérus handler. |
| `3` | `PROF_OAK` | Oak's dedicated handler. |
| `4` | `KURT` | Kurt's dedicated handler. |
| `5` | `BIKE_SHOP` | Bike Shop handler. |
| `6` | `KENJI` | Black Belt Kenji handler. |
| `7` | `BILL` | Bill handler. |
| `8` | `DAYCAREMAN` | Day-Care Man handler. |
| `9` | `DAYCARELADY` | Day-Care Lady handler. |
| `10` | `BUENA` | Buena handler. |
| `11` | `ETHAN_LYRA` | Childhood-friend handler. |
| `12` | `GYMLEADER` | Fighting Dojo/Gym Leader rematch handler. |
| `13` | `BAOBA` | Safari Zone Warden handler. |
| `14` | `IRWIN` | Juggler Irwin handler. |
| `15` | `UNK15` | Unnamed/unknown handler. |

Changing this byte changes which ARM handler reads the contact. For example, changing an ordinary trainer from 0 to 12 does not create a Gym Leader rematch: the Gym Leader handler expects Gym Leader message layouts and Fighting Dojo state.

### Title

The title label in the phone list is selected from the `trainerClass` text archive; or text archive `271`; or is empty.

| title (`trainerClass` in decomp) range/value | Source of the displayed label |
|---|---|
| `0`-`199` | Normal trainer-class formatter. Vanilla class-name data is populated only through class `128`; a hack using `129`-`199` must also supply normal class-name text. |
| `200` (`PHONE_MOM`) | Empty label. Mom displays no trainer class. |
| `201`-`207` | Message archive 271, starting at message 38 plus `trainerClass - 201`. |

The special values are:  
- 201 Pokemon Professor
- 202 Childhood Friend
- 203 Poke Ball Creator
- 204 Day-Care
- 205 Radio Personality
- 206 Poke Maniac
- 207 Safari Warden

Elm and Oak both display "Pokemon Professor", while Ethan/Lyra display "Childhood Friend" for example. The name itself still comes from message `0` of that contact's own phone text archive.

#### Compatibility with expanded trainer classes

<details>
<summary>Click for details...</summary>

This is a real compatibility limit for hacks that add trainer classes. The phone UI does not ask the normal trainer-class-name system whether a class is valid. Its logic is hardcoded as:

```text
if trainerClass == 200:
    show no class label                 # Mom
else if trainerClass >= 201:
    show message 38 + (trainerClass - 201) from archive 271
else:
    show the normal trainer-class name
```

Therefore the normal-class branch of an unpatched Pokegear phone UI accepts **0-199**, although vanilla class-name data exists only for `0`-`128`. Values 200-207 are reserved for non-trainer phone contacts, but the comparison actually catches **every value from 201 upward**.

:::caution  
If a hack adds normal trainer classes at 200 or above, changing the trainer-class text archive alone is insufficient.  
:::

A hypothetical trainer class **210** would be interpreted as a special phone title and reads message `38 + (210 - 201) = 47` from archive 271. It will display whatever message 47 contains; it does not gain a normal class name merely because a hack added class 210 elsewhere.

Separately, `trainerClass` is one unsigned byte. A phonebook row can therefore store only `0` through `255` as either a normal trainer-class ID or a special phone-contact title selector. Expanding normal trainer classes and moving the special values above them cannot extend this field beyond `255` without changing the phonebook record format and every reader of it.

</details>

### Trainer IDs for trainers and non-trainers

Vanilla non-trainer contacts use `TRAINER_NONE` (`0`), in the `trainerId` field: Mom, Elm, Oak, Ethan/Lyra, Kurt, the Day-Care couple, Buena, Bill, and the other special contacts do not carry a "real" trainer battle ID.

Trainer contacts carry their original battle ID, regardless of whether the contact can ever be rematched. The battle/rematch code decides whether that ID has a usable rematch row; the gift field and phone-item state do not require a rematch lookup. In the vanilla contact table there are 63 rows which do **not** have trainerId of `TRAINER_NONE`, matching the 63 overlay 26 rematch sets. Custom non-trainer contacts should use `TRAINER_NONE` (`0`) when no field trainer exists.

### What `mapId` controls

`mapId` has several proven uses:

- When a phone conversation is initialised, the location name of the header in `mapId` is stored in a message buffer (`3`), to be potentially used in conversations. The player's current location name is similarly stored in buffer `2`.
- Random incoming calls exclude a contact when the player is on the same `mapId`.
- Most special handlers and the generic handler use a same-map check to select their "local" response.
- Every type-0 (generic) phone call prepares two Pokemon-name message buffers before it prints its selected archive message: buffer `10` receives a random member of the caller's `trainerId` battle party, and buffer `11` receives a species selected from the wild-encounter data for this `mapId`. For buffer `11`, a contact whose `trainerClass` byte is exactly `11` (`0x0B`, Fisherman) selects one of the five Good Rod slots (with the night-fishing replacement at night); all other values select one of the twelve land slots for the current morning/day/night. A map with no wild encounters supplies Rattata instead. This is implemented by `GearPhoneCall_Generic` and `getRandomEncounterSlot` in Pokegear Overlay 101's generic phone-script code.
- If the contact's map-header ID is exactly `96` (`0x0060`, `MAP_NATIONAL_PARK`), the random incoming-call manager excludes that contact when flag `0x996` (`2454`, decomp name: `FLAG_UNK_996`) is set. The decomp calls its direct helper `Save_VarsFlags_CheckBugContestFlag`. The generic rematch/gift header conditions use the same check.
- The Pokegear map shows its pending battle marker when a **generic** (`type = 0`) contact at that `mapId` has either a valid ordinary rematch or a queued **dynamic** phone gift.

:::info  
The Pokegear phone book's location sort uses the stored location rank at `sortParam[2]`; it does not derive an order from `mapId` at runtime.
:::

---

## Registering phone numbers

The static phonebook table does not immediately and fully populate contacts in the player's Pokegear. Registration is saved separately in `SavePokegear.phoneContacts[]`. Empty slots are `FF`; registered slots contain contact IDs.

Normal field scripts use the following command IDs. The first name is the established scrcmd/DSPRE name; the second is the decomp name for the same command.

| Opcode | scrcmd / editor name | Decomp name | Use |
|---|---|---|---|
| `0x0092` | `RecordPokegearNumber contact` | `RegisterGearNumber contact` | Add a valid contact ID to the player's phonebook. |
| `0x0093` | `CheckPokegearNumberRegistered contact result` | `CheckRegisteredPhoneNumber contact result` | Write 0 or 1 depending on registration state. |

### Sorting phone numbers

There are a number of ways of sorting phone numbers in the Pokegear. The first three use a stored one-byte rank in each contact record, rather than recalculating from the displayed text or `mapId`:
- Title (`sortParam[0]`)
- Alphabet (`sortParam[1]`)
- Location (`sortParam[2]`)
- Manual arrangement

---

## Incoming and outgoing calls

Incoming and outgoing calls use different map permissions and different dialogue selection paths.

| Direction | Map-header field | Engine check | What it controls |
|---|---|---|---|
| Outgoing | `outgoingCalls` | `MapHeader_CanPlacePhoneCalls` | Whether the player can successfully start a call from the Pokegear. |
| Incoming | `incomingCalls` | `MapHeader_CanReceivePhoneCalls` | Whether random callers and queued event calls can ring on the current map. |

These are independent one-bit fields in `MapHeader`. A map can permit outgoing calls while blocking incoming calls.

DSPRE's map-header editor can set or clear `outgoingCalls` and `incomingCalls` independently.

### Outgoing

An outgoing call begins when the player selects a registered contact in the Pokegear. `outgoingCalls` must be enabled in the current map header. The selected contact's `type`, `mapId`, and, for type-0 contacts, the generic outgoing phone-script headers determine the conversation; this is distinct from the incoming random-call and queued-event paths below.

### Incoming: time trigger

The game keeps a runtime counter of elapsed **real-time minutes** for incoming calls. This timing gate is shared by queued system-event calls and ordinary random calls. While the player is in the normal overworld and no field task is running, completed minutes are added to this counter. At `10` minutes, the next normal input update when the player is standing still or has finished a movement checks for a call. It is not based on steps, frames, or a random per-step chance.

Queued system-event calls are checked first. Their triggering code can set the counter to one minute below the current threshold, making the queued call eligible after the next completed minute; this uses the threshold dynamically. A map which disallows incoming calls does not clear the counter; the check instead waits until the player reaches an allowed map. The timer is runtime state, so it is reset when a call is accepted or otherwise cleared.

#### Changing the shared incoming-call threshold (US HeartGold)

The vanilla ten-minute interval can be changed with two ARM9 immediate-byte edits. Both values must be changed to the same whole-minute value `N` (`1`-`255`). These are US HeartGold ARM9 **file offsets**; verify the original bytes before using them on another revision.

| ARM9 file offset | RAM address | Vanilla bytes | Meaning | Change for `N` minutes |
|:---:|:---:|---|---|---|
| `0x92DB2` | `0x02092DB2` | `0A 20` | Sets the call interval to `10` minutes. | Replace `0A` with `N`; retain `20`. |
| `0x92E44` | `0x02092E44` | `0A 29` | Compares the first elapsed-time update against `10` minutes. | Replace `0A` with `N`; retain `29`. |

For example, to use twenty minutes, change `0A 20` to `14 20` at `0x92DB2` and `0A 29` to `14 29` at `0x92E44`. The second edit keeps the first-update safeguard consistent with the interval. A queued trigger which sets the timer to one minute below the threshold still becomes eligible after one completed minute, because it uses the changed threshold value dynamically; a queued trigger which does not expedite itself waits for the shared threshold.

The shared incoming-call check:

1. Rejects maps where incoming calls are disabled.
2. Handles queued event calls first.
3. Continues to the ordinary random-call branch only when no queued call is started.

---

#### Incoming: random frequency

The following behaviour applies only to the ordinary random-call branch:

1. Makes a `51%` roll (`0`-`50` inclusive out of `0`-`99`). A failed roll clears the elapsed-minute counter, so the next ordinary attempt is after another ten minutes.
2. Chooses an `unkF` bucket: 0 (50%), 1 (30%), or 2 (20%).
3. Keeps only registered contacts whose `type` is `0` (generic trainer), `10` (Buena), `11` (Ethan/Lyra), `12` (Gym Leader), or `14` (Irwin). All other types are excluded from this ordinary random-caller pool, though they may still be called by the player, start an event call, or be started by a field script.
4. Removes contacts on the player's current map, contacts already recorded in that bucket's call history, Buena in unsuitable conditions, and National Park contacts during the Bug-Catching Contest.
5. Randomly chooses from the remaining contacts. If no candidate remains, it clears the elapsed-minute counter.

The call starts with `isScriptedCall = 0`. A type-0 contact uses the generic dialogue logic; types 10, 11, 12, and 14 each use their own dedicated handler.

##### Where the allowed-type list is defined

<details>
<summary>Click for details...</summary>

At US decompressed Overlay 2 file offset `0xC45C` (RAM `0x02251FDC`), the random-candidate builder contains this literal chain of five comparisons:

```text
type == 0 || type == 12 || type == 11 || type == 10 || type == 14
```

For the US decompressed Overlay 2, this function starts at **file offset `0xC45C`** (RAM `0x02251FDC`), using Overlay 2's RAM base `0x02245B80`. The comparisons are executable Thumb code, not five adjacent data bytes. Adding another allowed `type` therefore requires a binary code patch to this function, not an edit to the contact record alone.

</details>

##### What "recently called" means

<details>
<summary>Click for details...</summary>

The random-caller history is persistent save data, not a timer. It is the eight-byte `SAVE_MISC_DATA.unk_0280` field at structure offset `0x280`, with `FF` marking an unused slot. The game partitions it by `unkF` bucket:

| Chosen bucket | History bytes in `SAVE_MISC_DATA + 0x280` | Number of remembered contact IDs |
|:---:|---|:---:|
| `0` | `00`-`03` | `4` |
| `1` | `04`-`05` | `2` |
| `2` | `06`-`07` | `2` |

The candidate builder checks only the history range for the bucket chosen at step 3. A matching contact ID is excluded. If the history already contains every current eligible candidate in that bucket, it clears that bucket's history before filtering so that the pool cannot become empty forever. When the player accepts an ordinary random incoming call, the selected contact ID is appended to that bucket's range; if it is full, the oldest entry is discarded. Queued event and field-scripted calls do not use this history.

The filter is at US Overlay 2 file offset `0xC45C` / RAM `0x02251FDC`; the save helpers are identified by the decomp as `sub_0202AA44`, `sub_0202AA9C`, `sub_0202AAD4`, and `sub_0202AB18` in `src/save_misc.c`. The actual save-file location of the containing `SAVE_MISC_DATA` block is outside the scope of this page; use its structure-relative offset rather than assuming a fixed offset in every save file.

</details>

---

#### Incoming: queued system-event trigger

Queued event calls are not ordinary random calls and are not entries in the normal event-flag or variable systems. `PhoneCallPersistentState.callTriggerFlags[2]` is a separate two-byte packed bitfield in the phone-call persistent save data. Bits `0` through `12` are the 13 trigger IDs below.  

In vanilla, native game code sets these bits when it detects its own condition, such as an Egg hatching, cycling 1,024 steps, or a full PC. Field scripts can also set one with unnamed command `ScrCmd_148` (`0x0094`): it takes two bytes, `triggerId` (`0`-`12`) and `expedite` (`0` or nonzero). It sets that trigger bit; when `expedite` is nonzero, it advances the shared incoming-call timer to one minute before its threshold if necessary. `UnsetPhoneCallTrigger` (`0x0095`) takes one `triggerId` byte and clears that bit.  

The caller, predefined phone script, and pickup behaviour do not live in the bitfield. They come from a separate 13-record table in Overlay 2 (`ov02_02253C84` in `src/field/overlay_2_gear_phone.c`). Its RAM address `0x02253C84` converts to **file offset `0xE104`**. It occupies `0x4E` bytes: `0xE104` through `0xE151` inclusive. Record `n` is the definition for trigger bit `n`; its file offset is `0xE104 + (n * 6)`.

```text
callerId (u8), unknown (u8), phoneScriptId (u16 little-endian), forcePickUp (u8), unknown (u8)
```

| Bit | File offset | Trigger ID | Caller ID | Phone script ID | Condition that queues it | `forcePickUp` |
|:---:|:---:|---|:---:|:---:|---|:---:|
| `0` | `E104` | `ELM_EGG_HATCHED` | `1` Elm | `000D` | Gift Togepi Egg hatches. | `0` |
| `1` | `E10A` | `ELM_POKERUS` | `1` Elm | `0007` | Pokérus is discovered. | `0` |
| `2` | `E110` | `BIKE_SHOP_STEPS` | `15` Bike Shop | `0055` | 1,024 bicycling steps, before the Bike Shop call flag is set. | `1` |
| `3` | `E116` | `BILL_PC_FULL` | `9` Bill | `005D` | Every PC storage slot is full; the one-time Bill condition allows it. | `1` |
| `4` | `E11C` | `OAK_DEX_PROGRESS` | `2` Oak | `0000` | A National Dex ownership milestone is reached before its acknowledgement flag is set. | `0` |
| `5` | `E122` | `DAYCARE_HAS_EGG` | `6` Day-Care Man | `0000` | An egg is available and the Day-Care call flags allow it. | `0` |
| `6` | `E128` | `BAOBA_NEW_POKEMON` | `24` Baoba | `0000` | A new Safari Zone species/area arrangement is available. | `0` |
| `7` | `E12E` | `BAOBA_NEXT_TEST` | `24` Baoba | `008E` | Safari Zone test progression requests the next test. | `1` |
| `8` | `E134` | `BAOBA_OBJECT_ARRANGEMENT` | `24` Baoba | `008F` | Safari Zone object-arrangement progression. | `1` |
| `9` | `E13A` | `BAOBA_MORE_OBJECTS` | `24` Baoba | `0090` | Safari Zone more-objects progression. | `1` |
| `10` | `E140` | `BAOBA_EVEN_MORE_OBJECTS` | `24` Baoba | `0091` | Safari Zone even-more-objects progression. | `1` |
| `11` | `E146` | `BAOBA_MEMORY_LOSS` | `24` Baoba | `0092` | Safari Zone memory-loss progression. | `1` |
| `12` | `E14C` | `MOM_BOUGHT_SOMETHING` | `0` Mom | `001B` | Mom spends the player's saved money on an item. | `0` |

When a set trigger is selected, the ring manager reads its matching table record, assigns the caller and `phoneScriptId`, and marks the call as an event call (`isScriptedCall = 3`). The `forcePickUp` byte decides what happens next:

| `forcePickUp` | Result |
|:---:|---|
| `0` | The phone rings normally. The player opens/answers the Pokegear phone as usual. |
| `1` | The game begins field scene script `0x7FF` immediately. The player does not get a normal opportunity to ignore the ring. |

The table above accounts for all 13 triggers. Its `forcePickUp = 1` rows are Bike Shop, Bill, and the five Baoba progression calls; the other six ring normally. In the US file, both unknown bytes are `00` in every record; their purpose is not established, so leave them unchanged.

:::warning  
These offsets and byte values are for the US vanilla *decompressed* Overlay 2. Another version or language must be mapped independently.  
:::

:::caution No in-place space for a fourteenth event-call record  
The 13-record table ends at US decompressed Overlay 2 offset `0xE151`, but the six following zero-looking bytes are **not** free space. The Celebi cutscene's live `VecFx32 { 0, 0x1000, 0 }` constant begins at `0xE154` and occupies `0xE154`-`0xE15F`; it is used by `CelebiCutscene_SwirlEffect` in `src/field/event_cutscene_celebi.c`. Its first `u32` is zero, which makes `0xE154`-`0xE157` appear unused in a hex editor. Writing record 13 at `0xE152` would overwrite that vector's `x` component.

Only `0xE152`-`0xE153` are alignment padding. A real fourteenth record requires relocating or otherwise expanding the table and repointing its Overlay 2 readers; this page does not yet provide that relocation patch.  
:::

#### Accepting trigger ID 13 (US version)

The existing two-byte `callTriggerFlags` field can store bit `13`; it does not require a save-format change. Before a relocated record 13 can work, six immediate-byte edits must change the valid trigger count from `13` (`0D`) to `14` (`0E`). These edits make `SetPhoneCallTrigger 13, ...` set the bit, make the incoming selector scan it, and make answering the call clear it.

| File | File offset | Vanilla bytes | Replacement bytes | Purpose |
|---|:---:|---|---|---|
| ARM9 | `0x2F01E` | `0D 29` | `0E 29` | Accept trigger ID `13` in the persistent-bit setter. |
| ARM9 | `0x2F052` | `0D 29` | `0E 29` | Accept trigger ID `13` in the persistent-bit clearer. |
| ARM9 | `0x2F08E` | `0D 29` | `0E 29` | Read trigger bit `13` in the persistent-bit checker. |
| Overlay 2 | `0xC6A2` | `0D 21` | `0E 21` | Allocate 14 bytes for queued trigger candidates. |
| Overlay 2 | `0xC6AC` | `0D 22` | `0E 22` | Clear all 14 candidate bytes. |
| Overlay 2 | `0xC6FA` | `0D 2C` | `0E 2C` | Scan trigger IDs `0` through `13`. |

The ARM9 comparisons are the set, clear, and check helpers in `src/save_pokegear.c`; the Overlay 2 values are in `ov02_02252218` in `src/field/overlay_2_gear_phone.c`. The script command `0x0094` already accepts its trigger ID as a byte, so no command-format change is required.

**Test after the table itself has been safely relocated:** define record 13 at its new location, use `0x0094` with `triggerId = 13` and a nonzero expedite byte, then wait on an incoming-call-enabled map. The intended caller must ring, answering must clear the bit, and the same call must not recur after save/reload. This test proves the new index's set, scan, and clear paths; it does not validate a table relocation on its own.

#### Vanilla trigger setters (decomp references)

<details>
<summary>Click for details...</summary>

The event-record table above defines what each pending bit does. This table instead records every vanilla path which **sets** a bit. It is a source map for advanced hackers: most paths call `sub_02092E14`, which sets the persistent bit and, when its final argument is true, advances the shared incoming-call timer. Bit `1` is set by a field script, and bit `6` writes the persistent bit directly.

| Bit | Vanilla setter and condition | Expedites the incoming timer? |
|:---:|---|:---:|
| `0` | `sub_02093134` in `src/unk_02092BE8.c`, called by `src/hatch_egg_task.c` after an egg hatches. It sets the bit only when `MonIsFromTogepiEgg` identifies the special Elm Egg. | Yes |
| `1` | New Bark's field script, `files/fielddata/script/scr_seq/scr_seq_0003.s`, uses `ScrCmd_148 CALL_TRIGGER_ELM_POKERUS, FALSE` after its Pokerus check. This is the vanilla example of a script setting a queued-call bit. | No |
| `2` | `FieldSystem_UpdateBikeShop` in `src/field/field_control.c`: the Bike Shop call has not already happened, bit `2` is not already pending, and `GAME_STAT_STEPS_BIKED` is at least `1024`. | Yes |
| `3` | `sub_02093070` in `src/unk_02092BE8.c`: Bill is registered, the full-PC call flag is clear, and every PC storage slot is full. This helper is called after several battle and storage-related return paths. | Yes |
| `4` | `sub_020930C4` in `src/unk_02092BE8.c`: Oak is registered, the National Dex owned count has reached a nonzero multiple of `50`, and the corresponding Oak acknowledgement flag is clear. | No |
| `5` | `sub_0209316C` in `src/unk_02092BE8.c`, called when the Day-Care generates an Egg in `src/get_egg.c`: the Day-Care Man is registered, and the script-flag guard permits the Egg call. | Yes |
| `6` | `SaveData_SafariZone_CheckAreasWithUpdatedEncounters` in `src/unk_02097268.c`: after at least one day has passed outside an active Safari session, it compares the Safari Zone's encounter lists before and after the daily object-level update. It sets bit `6` if an area now has changed encounters, and clears it when none do. | No |
| `7` | `sub_02092E54` in `src/unk_02092BE8.c`: no Baoba bits `7`-`11` are already pending, Safari progress variable `VAR_UNK_4057` is `3`, and the Safari Zone's three-hour timer has elapsed. | Yes |
| `8` | `sub_02092E54`: Safari progress is at least `6`, the player has the National Dex, the three-hour timer reports its ordinary elapsed state, and the Safari object-unlock level is `0`. | Yes |
| `9` | `sub_02092E54`: the same post-National-Dex three-hour branch, with object-unlock level `1` or `2`. | Yes |
| `10` | `sub_02092E54`: the same branch, with object-unlock level `3`; it is also chosen by the routine's exceptional elapsed-time result when the unlock level is at least `3`. | Yes |
| `11` | `sub_02092E54`: the routine's exceptional elapsed-time result when the object-unlock level is below `3`. | Yes |
| `12` | `src/battle/battle_setup.c` calls `MomGift_TryEnqueueGiftOnBalanceChange` after Mom's savings balance changes. If that helper successfully queues a purchase, it sets bit `12`. The purchase and its quantity are stored in Mom's separate gift queue. | Yes |

`VAR_UNK_4057` is a normal script variable read by the Baoba evaluator; it is distinct from the queued-call bitfield. The three-hour test is `sub_0202F798` in `src/safari_zone.c`. The exact story meaning of every `VAR_UNK_4057` value is not yet named by the decomp, so the table deliberately states the proven values and conditions rather than assigning unsupported labels to them.

</details>

Mom's purchase call and the item collection are separate: the queued call announces the purchase, while the home field script retrieves the stored item, checks Bag space, and clears `CALL_TRIGGER_MOM_BOUGHT_SOMETHING` only after a successful award.

---

### Incoming: field-script trigger

Not every incoming call uses the queued trigger-bit table. A normal field script can configure and immediately launch a phone call with these commands. Elm's stolen-Pokemon call is a vanilla direct field-scripted call: it selects `PHONE_SCRIPT_002` and does not set or consume any queued event-trigger bit.

| Opcode | scrcmd / editor name | Source name | Use |
|---|---|---|---|
| `0x01AE` | `SetPhoneCall contact scriptedFlag predefinedScript` | `SetPhoneCall` | Configure a field-scripted phone call. |
| `0x01AF` | `RunPhoneCall` | `RunPhoneCall` | Launch the configured phone call. |

---

## Phone scripts and text archives

### What a phone conversation is made from

The term "phone script" does **not** mean a normal map field script. A normal field script is bytecode that runs on a map, such as the stolen-Pokemon sequence; it can register a number or start a call. A phone conversation is assembled from compact Pokegear data and the contact's message archive.

| Part | What it controls | Where it lives |
|---|---|---|
| Field script | Map events, including direct calls through `0x01AE` and `0x01AF`. | Map script data. |
| Generic header | Whether one ordinary trainer-call outcome is eligible. | Overlay 101. |
| Phone-script definition | The selected message IDs and any result, such as a rematch or gift. | Overlay 101. |
| Contact text archive | Contact name and the actual words printed by messages. | Message NARC. |

The following data flow applies to an ordinary **type-0** contact:

```text
contact and call direction
  -> that contact's outgoing or incoming header rows
  -> first eligible header chooses a phone-script definition ID
  -> definition chooses a message and optional side effect
  -> message ID is read from that contact's text archive
```

This separation matters when editing. A header chooses an outcome; a definition describes that outcome; a text archive supplies its wording.

### Generic phone calls: selection flow

For a type-0 contact, the call code chooses the outgoing or incoming half of the contact's 16 generic-header rows. It checks up to eight rows in order. A matching row supplies a phone-script definition ID. For an outgoing call made while standing on the contact's own `mapId`, the generic handler instead uses the contact record's `phoneScriptIfLocal` value.

The definition selects one message ID for a male player and one for a female player, then optionally applies a result. Known generic results include ordinary dialogue, a pending rematch, a dynamic phone gift, a save-flag operation, and a random inserted word or phrase.

### Generic phone-call data in Overlay 101

The generic tables are compiled into **Overlay 101**, the Pokegear application overlay. They are not in `pmtel_book.dat`, a contact text archive, or a map-script NARC. For the US decompressed `overlay_0101.bin`, Overlay 101 loads at `0x021E7740`.

| Data | US Overlay 101 file range | US RAM range | Addressing rule |
|---|---|---|---|
| Phone-script definitions | `1143C`-`11EEB` | `021F8B7C`-`021F962B` | Definition ID `s`: `1143C + (s * 6)`. IDs `0`-`455`; ID 0 is the all-zero no-script definition. |
| Greeting message IDs | `11EEC`-`11F4B` | `021F962C`-`021F968B` | Eight rows of twelve one-byte message IDs, separate from conversation/result definitions. |
| Generic call headers | `11F4C`-`13B6B` | `021F968C`-`021FB2AB` | Contact ID `c`: `11F4C + (c * 60)`. Each 16-row block is `0x60` bytes. |

All offsets in this section are hexadecimal and apply only to the **US decompressed** Overlay 101. Do not use them in a compressed overlay or another revision without remapping from that overlay's RAM base.

#### Header records: choosing an outcome

Each generic header is six bytes. The first eight rows are outgoing-call choices; the next eight are incoming-call choices.

```text
byte 0: generic condition type
byte 1: chance
bytes 2-3: little-endian header `scriptType` field
bytes 4-5: little-endian phone-script definition ID
```

The decomp names the middle word `scriptType`. It is a playback-handler index: `0`, `1`, and `2` select the simple, species-buffering generic, and random-line handlers; higher values select specialised handlers. It must not be confused with the side-effect type stored in a phone-script definition.

| Direction | Header rows | First-row offset for contact `c` |
|---|---|---|
| Outgoing | `0`-`7` | `11F4C + (c * 60)` |
| Incoming | `8`-`15` | `11F7C + (c * 60)` |

`PHONECALLGENERIC_NIL` (`0`) and `PHONECALLGENERIC_NONE` (`FF`) stop the header scan. They are control rows, not ordinary dialogue outcomes. Do not insert or delete header bytes: later rows are addressed at fixed offsets.

#### Script-definition records: message and side effect

Each phone-script definition is six bytes. Its first two bytes are message IDs in the **current caller's** text archive, selected by the player's gender.

```text
byte 0: message ID for a male player
byte 1: message ID for a female player
bytes 2-3: little-endian word: low 4 bits = side-effect type; upper 12 bits = parameter 0
bytes 4-5: little-endian parameter 1
```

| Low-nibble value | Decomp name | Result |
|:---:|---|---|
| `0` | `NONE` | Prints the selected message with no known persistent phone result. |
| `1` | `UNK1` | Sets or clears a save flag. Its observed implementation is the same as type `2`. |
| `2` | `FLAG` | Sets or clears a save flag. |
| `3` | `REMATCH` | Sets the caller's pending rematch state. |
| `4` | `ITEM` | Queues a dynamic phone gift for collection from the NPC. |
| `5` | `WORD` | Inserts a random word or message fragment. |

The parameter meanings for every individual `ITEM`, `FLAG`, and `WORD` definition are not all documented. Do not alter a non-`NONE` definition's packed fields only because its text looks suitable.

### Contact text archives

Each contact ID maps to one phone message archive through `sPhoneMessageGmm` in `phonebook_dat.c`; Mom, for example, maps to `NARC_msg_msg_0664_bin`. Message **0** in that archive is the contact name displayed by the phone-call UI.

The selected phone-script definition provides the next message ID, which is read from that same contact archive. Therefore one definition ID can produce different wording when used by another contact: its message numbers are reinterpreted against that other contact's archive.

`GetPhoneContactMsgIds` returns the contact archive and an input message number plus one. It does not choose generic versus special dialogue and does not select the caller; those decisions have already been made by the header, special handler, queued-event record, or field script.

### Calls that do not use generic headers

The generic header route is only for type-0 contacts. Mom, Elm, Oak, Gym Leaders, Baoba, and other special contact types use dedicated code to choose their call behaviour. A queued event-call record in Overlay 2 already supplies a predefined phone-script ID, while a field script can configure and launch a direct call through `SetPhoneCall` and `RunPhoneCall`.

Changing a generic header does not change these special, queued, or field-scripted calls.

### Editing existing generic calls

| Goal | Edit | Behavioural risk | Independent test |
|---|---|---|---|
| Change wording only | Edit the selected message in the contact's text archive. | None expected. | Trigger the same call; only the wording changes. |
| Change one incoming outcome | Repoint that incoming header's two-byte definition ID to an existing `NONE` definition. | The replacement definition's message IDs are interpreted in the caller's archive. | Trigger that outcome; other incoming outcomes remain available. |
| Remove a rematch offer | Repoint its header to a known ordinary-dialogue definition, leaving the original definition intact for other users. | Prevents the `seeking` side effect only for that header. | Confirm no map marker or rematch battle appears after the call. |
| Add a definition or condition | Requires verified free Overlay 101 space, a valid new ID, and further code tracing. | Not yet a documented safe binary edit. | Do not treat an appended six-byte record as usable without that work. |

### Advanced reference: code and raw records

<details>
<summary>Click for details...</summary>

The decomp provides these names as a readable source map for the binary logic:

| Decomp function | Source file | Role |
|---|---|---|
| `PhoneCall_GetScriptId_Generic` | `scripts/phone_scripts_generic.c` | Calculates `callerID * 16`, chooses the outgoing or incoming group, and returns the selected definition ID. |
| `PhoneScriptGeneric_GetScriptIdInternal` | `scripts/phone_scripts_generic.c` | Walks at most eight headers, stops at `NIL` or `NONE`, evaluates conditions, and returns a definition ID. |
| `PhoneCall_GetScriptDefPtrByID` | `overlay_101_021F1D74.c` | Resolves `gPhoneCallScriptDef[scriptID]`. |
| `PhoneCall_ApplyGenericNPCcallSideEffect` | `overlay_101_021F1D74.c` | Interprets `REMATCH`, `ITEM`, `UNK1`, `FLAG`, and `WORD` definition side effects. |

The first definition records are `00 00 00 00 00 00` for ID 0, `01 02 00 00 00 00` for `PHONE_SCRIPT_001`, and `21 22 00 00 00 00` for `PHONE_SCRIPT_002`. These first scripts are Elm messages; copying one to another contact can point at different text because the message archive changes.

The first header is `00 64 00 00 00 00`: `NIL`, chance `100`, and definition ID 0. The following unused header is `FF 00 00 00 00 00`: `NONE`.

The compiled function entries for the generic selector and definition interpreter have not yet been independently mapped in US Overlay 101 and should not be guessed from a decomp source filename. The handler-dispatch table itself is mapped: US Overlay 101 file offset `0x10F3C` contains Thumb pointers. Handler indices `0`, `1`, and `2` point to `0x021F2681`, `0x021F2F51`, and `0x021F2FFD` respectively.

</details>

---

## Trainer rematches

### Ordinary phone-trainer rematches

Ordinary trainer rematches use both phone state and a fixed trainer-ID table in the **US vanilla Overlay 26** at **file offset `0x20C`**. It has 63 rows of 12 bytes (`0x20C` through `0x4FF` inclusive). Each record is six little-endian `u16` trainer battle IDs. The six entries relate to the original trainer ID (used to lookup from the `pmtel_book.dat` phonebook data) and five possible rematch teams, each unlocked with the setting of an individual flag.

| Entry | Relative offset | Tier unlock flag | Meaning |
|:---:|:---:|---|---|
| `0` | `00` | None | Original/base battle ID. It identifies the row and is the selector’s final fallback. |
| `1` | `02` | `0x97C` | Duplicate base battle ID. Vanilla never sets this flag. It is a redundant positive-index fallback: selecting it produces the same battle as entry `0`. |
| `2` | `04` | `0x97D` | First distinct rematch team. Set by the Radio Tower story sequence. |
| `3` | `06` | `0x97E` | Second distinct rematch team. Set by the Hall of Fame sequence. |
| `4` | `08` | `0x97F` | Third distinct rematch team. Set when the player receives the Earth Badge at Viridian Gym. |
| `5` | `0A` | `0x980` | `TRAINER_NONE` (`0x0000`) in every vanilla ordinary-trainer row. It is a terminator in vanilla, but can hold a fourth distinct rematch team in a hack. |

An **entry** is a physical slot in a table row. A **rematch tier** is an entry used as a progressively unlocked battle team: entries `1` through `5` can serve as tiers, while entry `0` is the base battle and fallback. The decomp calls the corresponding flag check a “rematch group”; there is no separate tier data structure.

Rows `0`–`46` are ordinary trainers; rows `47`–`62` are Gym Leaders.

<details>
<summary>Vanilla Overlay 26 table</summary>

> Research: [pret/pokeheartgold](https://github.com/pret/pokeheartgold), Eclipse (Luna).  

| Row | Overlay 26 offset | Trainer | Entry 0 | Entry 1 | Entry 2 | Entry 3 | Entry 4 | Entry 5 |
|:---:|---|---|---|---|---|---|---|---|
| 0 | `0x020C` | Bird Keeper Jose | `414` | `414` | `303` | `446` | `602` | `0` |
| 1 | `0x0218` | Picnicker Erin | `151` | `151` | `335` | `453` | `603` | `0` |
| 2 | `0x0224` | Picnicker Liz | `27` | `27` | `276` | `277` | `518` | `0` |
| 3 | `0x0230` | School Kid M Chad | `397` | `397` | `434` | `435` | `507` | `0` |
| 4 | `0x023C` | Sailor Huey | `211` | `211` | `440` | `441` | `509` | `0` |
| 5 | `0x0248` | Bug Catcher Wade | `4` | `4` | `461` | `460` | `512` | `0` |
| 6 | `0x0254` | Youngster Joey | `8` | `8` | `279` | `280` | `510` | `0` |
| 7 | `0x0260` | School Kid M Jack | `178` | `178` | `430` | `431` | `503` | `0` |
| 8 | `0x026C` | Ace Trainer M Gaven | `102` | `102` | `456` | `457` | `604` | `0` |
| 9 | `0x0278` | Black Belt Kenji | `17` | `17` | `250` | `278` | `605` | `0` |
| 10 | `0x0284` | Hiker Parry | `145` | `145` | `451` | `452` | `606` | `0` |
| 11 | `0x0290` | Picnicker Tiffany | `402` | `402` | `466` | `467` | `522` | `0` |
| 12 | `0x029C` | Hiker Anthony | `61` | `61` | `100` | `155` | `523` | `0` |
| 13 | `0x02A8` | Ace Trainer F Reena | `114` | `114` | `444` | `445` | `607` | `0` |
| 14 | `0x02B4` | Fisherman Wilton | `124` | `124` | `325` | `450` | `608` | `0` |
| 15 | `0x02C0` | Ace Trainer F Jamie | `113` | `113` | `458` | `459` | `609` | `0` |
| 16 | `0x02CC` | Juggler Irwin | `7` | `7` | `454` | `455` | `527` | `0` |
| 17 | `0x02D8` | Poké Maniac Brent | `131` | `131` | `172` | `173` | `530` | `0` |
| 18 | `0x02E4` | School Kid M Alan | `24` | `24` | `432` | `433` | `505` | `0` |
| 19 | `0x02F0` | Pokéfan M Derek | `44` | `44` | `438` | `439` | `610` | `0` |
| 20 | `0x02FC` | Picnicker Gina | `65` | `65` | `142` | `334` | `520` | `0` |
| 21 | `0x0308` | Fisherman Tully | `123` | `123` | `323` | `324` | `517` | `0` |
| 22 | `0x0314` | Pokéfan F Beverly | `182` | `182` | `436` | `437` | `611` | `0` |
| 23 | `0x0320` | Bird Keeper Vance | `137` | `137` | `447` | `448` | `612` | `0` |
| 24 | `0x032C` | Fisherman Ralph | `57` | `57` | `462` | `463` | `515` | `0` |
| 25 | `0x0338` | Camper Todd | `66` | `66` | `274` | `275` | `525` | `0` |
| 26 | `0x0344` | Bug Catcher Arnie | `78` | `78` | `360` | `449` | `513` | `0` |
| 27 | `0x0350` | Lass Dana | `400` | `400` | `464` | `465` | `528` | `0` |
| 28 | `0x035C` | Lass Krise | `184` | `184` | `613` | `614` | `615` | `0` |
| 29 | `0x0368` | Youngster Ian | `64` | `64` | `616` | `617` | `618` | `0` |
| 30 | `0x0374` | Firebreather Walt | `388` | `388` | `619` | `620` | `621` | `0` |
| 31 | `0x0380` | Bug Catcher Doug | `140` | `140` | `622` | `623` | `624` | `0` |
| 32 | `0x038C` | Bug Catcher Rob | `48` | `48` | `625` | `626` | `627` | `0` |
| 33 | `0x0398` | Biker Reese | `313` | `313` | `628` | `629` | `630` | `0` |
| 34 | `0x03A4` | Biker Aiden | `574` | `574` | `631` | `632` | `633` | `0` |
| 35 | `0x03B0` | Biker Ernest | `579` | `579` | `634` | `635` | `636` | `0` |
| 36 | `0x03BC` | Teacher Hillary | `382` | `382` | `637` | `638` | `639` | `0` |
| 37 | `0x03C8` | School Kid M Billy | `331` | `331` | `640` | `641` | `642` | `0` |
| 38 | `0x03D4` | Twins Kay & Tia | `569` | `569` | `643` | `644` | `645` | `0` |
| 39 | `0x03E0` | Bird Keeper Josh | `565` | `565` | `646` | `647` | `648` | `0` |
| 40 | `0x03EC` | School Kid M Torin | `567` | `567` | `649` | `650` | `651` | `0` |
| 41 | `0x03F8` | Young Couple Tim & Sue | `559` | `559` | `652` | `653` | `654` | `0` |
| 42 | `0x0404` | Hiker Kenny | `358` | `358` | `655` | `656` | `657` | `0` |
| 43 | `0x0410` | Camper Tanner | `561` | `561` | `658` | `659` | `660` | `0` |
| 44 | `0x041C` | Fisherman Kyle | `59` | `59` | `661` | `662` | `663` | `0` |
| 45 | `0x0428` | Fisherman Kyler | `558` | `558` | `664` | `665` | `666` | `0` |
| 46 | `0x0434` | Gentleman Alfred | `401` | `401` | `672` | `673` | `674` | `0` |
| 47 | `0x0440` | Gym Leader Falkner | `20` | `20` | `712` | `0` | `0` | `0` |
| 48 | `0x044C` | Gym Leader Bugsy | `21` | `21` | `713` | `0` | `0` | `0` |
| 49 | `0x0458` | Gym Leader Whitney | `30` | `30` | `714` | `0` | `0` | `0` |
| 50 | `0x0464` | Gym Leader Morty | `31` | `31` | `715` | `0` | `0` | `0` |
| 51 | `0x0470` | Gym Leader Jasmine | `33` | `33` | `717` | `0` | `0` | `0` |
| 52 | `0x047C` | Gym Leader Chuck | `34` | `34` | `718` | `0` | `0` | `0` |
| 53 | `0x0488` | Gym Leader Pryce | `32` | `32` | `716` | `0` | `0` | `0` |
| 54 | `0x0494` | Gym Leader Clair | `35` | `35` | `719` | `0` | `0` | `0` |
| 55 | `0x04A0` | Gym Leader Brock | `253` | `253` | `720` | `0` | `0` | `0` |
| 56 | `0x04AC` | Gym Leader Misty | `254` | `254` | `721` | `0` | `0` | `0` |
| 57 | `0x04B8` | Gym Leader Lt. Surge | `255` | `255` | `722` | `0` | `0` | `0` |
| 58 | `0x04C4` | Gym Leader Erika | `256` | `256` | `723` | `0` | `0` | `0` |
| 59 | `0x04D0` | Gym Leader Janine | `257` | `257` | `724` | `0` | `0` | `0` |
| 60 | `0x04DC` | Gym Leader Sabrina | `258` | `258` | `725` | `0` | `0` | `0` |
| 61 | `0x04E8` | Gym Leader Blaine | `259` | `259` | `726` | `0` | `0` | `0` |
| 62 | `0x04F4` | Gym Leader Blue | `261` | `261` | `727` | `0` | `0` | `0` |

</details>

#### Rematch flow

After the relevant phone script executes, the game selects the trainer battle as follows:

1. The phone script sets `PhoneRematch.seeking` for the contact.
2. The trainer battle lookup checks that bit and uses the contact's `trainerId` from `pmtel_book.dat` to find its Overlay 26 row.
3. Starting at entry `1`, it checks entries in order:
   - If an entry contains `TRAINER_NONE` (`0`), it selects the immediately preceding entry and stops. This can select entry `0`: `TRAINER_NONE` in entry `1` selects entry `0`; `TRAINER_NONE` in entry `5` selects entry `4`.
   - Otherwise, if that trainer battle has not been beaten, it selects that entry and stops.
   - Otherwise, it continues to the next entry.
   - If it reaches the end without either condition, it selects entry `5`.
4. If the selected entry is `1` through `5`, the game checks its tier-unlock flag: `0x97B + entry index`. If that flag is clear, it returns the previous entry without rechecking whether that earlier battle has been beaten or whether its own tier flag is set. If no earlier rematch entry exists, it returns entry `0`.
5. The shared field script starts the returned trainer battle even if that battle has already been beaten. After a win, it sets that battle's trainer flag and clears `PhoneRematch.seeking`.

This means rematch teams can be fought repeatedly. Trainer flags normally move selection towards the next team, but they do not prevent a selected fallback team from being fought again after another qualifying phone call.

Entry `0` is therefore a real fallback, not only the row's lookup ID. For example, a custom unbeaten entry `1` with `0x97C` clear falls back to entry `0`.

In vanilla, entry `1` duplicates the base battle and entry `5` is `TRAINER_NONE`. Once entry `4` has been beaten, the scan reaches entry `5`, selects entry `4`, and allows that final upgraded team to be fought repeatedly.

:::caution
`0xFFFF` is not a safe placeholder in a rematch row. The selector skips it only while searching for an unbeaten candidate or walking backwards for an unlocked fallback. It does not skip it when a later `TRAINER_NONE` (`0`) terminator selects the preceding entry, and it does not validate the resulting trainer ID. Do not use `0xFFFF` as a repeat marker or immediately before a `TRAINER_NONE` terminator.
:::

### Gym Leader rematches

Gym Leaders use a dedicated phone handler to offer rematches, but the pending battle is selected from their Overlay 26 row by the same rematch-table lookup used for ordinary trainers. In vanilla, each Gym Leader row has the base battle in entries `0` and `1`, the first upgraded battle in entry `2`, and `TRAINER_NONE` in entries `3`–`5`.

An outgoing Gym Leader call checks all 16 badges, no existing pending rematch, and the contact's weekday/time-of-day fields before offering the battle. The player must also accept the offer before the handler sets `seeking`. Editing a Gym Leader row can change or extend its battle tiers; it does not alter those rematch availability conditions. Conversely, the ordinary generic `0x0C6` availability condition does not control Gym Leader rematches.

### Extending Rematch Tiers

The rematch-table mechanics apply to ordinary phone trainers and Gym Leaders: once a rematch has been offered, the matching Overlay 26 row supplies its battle ID. These changes alter battle-team progression only; they do not alter the separate rematch availability conditions.

#### Add a later tier: entry `5`

Replace `TRAINER_NONE` in entry `5` with a new trainer battle ID, then set tier flag `0x980` from a field script at the desired later milestone (after defeating Trainer Red, for example). Once entry `4` has been beaten, the selector can use the new entry `5` team. This does not require a new table or a larger row.

For an ordinary trainer row, vanilla already supplies entries `1`–`4`. For a Gym Leader row, vanilla entries `3`–`5` are `TRAINER_NONE`; populate every preceding entry that the selector must pass, because the first `TRAINER_NONE` ends its scan. The Gym Leader's separate offer requirements still apply.

#### Add an earlier tier: entry `1`

Replace the duplicate base battle ID in entry `1` with a new trainer battle ID, then set tier flag `0x97C` from a field script at the desired milestone. After the base battle has been beaten, the selector uses this new team when `0x97C` is set; before then, it falls back to entry `0`. This changes the rematch progression only. For Gym Leaders, it does not bypass their separate rematch availability conditions: 16 badges, schedule, and pending state.

### Rematch availability conditions

Rematch availability conditions decide whether a phone call can set `PhoneRematch.seeking`. They are separate from tier unlocks, which decide which battle is selected after `seeking` is set.

Ordinary trainers use different conditions depending on who starts the call:

| Direction | How a new rematch is offered | Requirements |
|---|---|---|
| Player calls trainer | A scheduled offer | Radio Tower cleared; matching weekday and time of day; chance roll. |
| Trainer calls player | A random offer | Radio Tower cleared; chance roll; no weekday or time check. |

The player-call route applies when the trainer is on a different map. A call to a trainer on the current map uses that contact's local call script instead. If a rematch is already pending, the ordinary-trainer phone scripts select an existing-rematch message rather than making another offer.

Weekday and time are therefore not universal rematch requirements: they apply to ordinary trainers only when the player initiates the new offer. Gym Leaders are a separate case; their outgoing handler also checks those fields, as described above.

#### Radio Tower availability condition (ordinary contacts only)

This subsection applies only to ordinary generic contacts, not Gym Leaders. The generic phone-condition code tests `FLAG_BEAT_RADIO_TOWER_ROCKETS` (`0x0C6`) before it permits relevant rematch calls. This is independent of rematch-tier flags. A hack may leave this availability condition at its vanilla point, change the phone-condition tests to another flag, or deliberately align it with `0x97C`.

The Radio Tower availability condition is `FLAG_BEAT_RADIO_TOWER_ROCKETS` (`0x0C6`). It is distinct from rematch-tier flag `0x97D`, which controls whether the Overlay 26 selector can use vanilla entry `2`. The Radio Tower completion sequence sets both flags, but the generic header checks `0x0C6` and the battle-team selector checks `0x97D` separately.

The generic `FIGHTPOSTROCKETS` condition also prevents a generic rematch script from being selected before the Radio Tower flag is set. Unlike `POSTROCKETWEEKTIME`, it does not read the phonebook record's weekday or time bytes. Other headers prevent another offer while `PhoneRematch.seeking` is already set, prevent calls while a dynamic gift is waiting, and exclude National Park during the Bug-Catching Contest. These are separate rematch availability conditions, so changing only the two schedule bytes does not remove them.

The Radio Tower flag is tested at three separate places in the **US vanilla decompressed Overlay 101**. The file offsets below are relative to `overlay_0101.bin`; its load address is `0x021E7740`.

| Header condition | What the condition adds | Function entry | `0x0C6` load |
| --- | --- | --- | --- |
| `FIGHTPOSTROCKETS` | Rejects a rematch script before Radio Tower; also rejects a caller already seeking a rematch or holding a dynamic gift. | `0xB578` (RAM `0x021F2CB8`) | `0xB5A4` |
| `FIGHTPOSTROCKETSNOTBUGCONTEST` | Requires Radio Tower, rejects National Park during an active Bug-Catching Contest, and requires an already-pending rematch. | `0xB608` (RAM `0x021F2D48`) | `0xB612` |
| `POSTROCKETWEEKTIME` | Requires Radio Tower, then checks the contact's weekday and time-of-day bytes. | `0xB5D0` (RAM `0x021F2D10`) | `0xB5D8` |

Overlay 101 calls these three conditions separately at `0xB4C4`, `0xB4CE`, and `0xB4E2`. Each condition loads `0x0C6` with the Thumb instruction `MOVS R1, #0xC6`, so patch every condition whose availability rule should change. A replacement flag from `0x0000` through `0x00FF` can use the existing one-byte immediate; `0x0100` or above, including `0x97C`, requires code that loads a full 16-bit flag ID. Do not move the normal setter for `0x0C6`: the flag also controls unrelated game behaviour.

The following existing low-byte milestone flags can be substituted as read-only availability conditions:

| Vanilla milestone flag | Immediate byte | Replacement availability condition first passes after |
|---|:---:|---|
| `FLAG_GOT_POKEGEAR` (`0x09C`) | `9C` | Receiving the Pokegear. |
| `FLAG_GOT_EGG_FROM_ELMS_ASSISTANT` (`0x070`) | `70` | Receiving the Togepi Egg; Route 32 progression is blocked until this flag is set. |
| `FLAG_GOT_TM30_FROM_MORTY` (`0x0A6`) | `A6` | Beating Morty and receiving TM30. |
| `FLAG_GOT_HM03` (`0x0A2`) | `A2` | Receiving HM03 after the Kimono Girls, when Surf travel becomes available. |

These are existing vanilla flags, not free flags for new state. They are suitable only as read-only milestone conditions; reusing or changing their normal setters would affect their original game behaviour. Changing this availability condition also does not alter tier unlocks or create trainer rematch teams; the applicable weekday/time, chance, and pending-state conditions still apply.

---

## Gifts after calls and rematches

There are two item systems that are easy to confuse:

| Gift path | Storage | Retrieval |
|---|---|---|
| Contact-configured rematch reward | `PhoneBookEntry.gift` | The post-rematch battle field script clears the pending rematch, calls `GetPhoneContactRandomGiftBerry`, and awards the returned item if it is nonzero. `ITEM_CHERI_BERRY` is a special sentinel: it returns one of the first ten berries at random. |
| Dynamic phone-queued gift | `PhoneRematch.giftItem` | A `PHONESCRIPTTYPE_ITEM` outcome resolves its item and stores the specific result here. `ITEM_CHERI_BERRY` becomes one of the first ten berries; `ITEM_POKE_BALL` becomes an Ultra Ball, Great Ball, or Poké Ball. The retrieval command clears the queued item immediately after reading it. |

The first path is not a generic “static item” given merely because a contact exists. In vanilla it is the reward path after a trainer's rematch battle. The nonzero rows are Joey (HP Up), Kenji (PP Up), Huey (Protein), Vance (Carbos), Parry (Iron), Erin (Calcium), and Ian (random berry via the Cheri sentinel).

The second path is the phone-only item offer system. Because the NPC field script checks for an available rematch before calling `GetPhoneContactGiftItem`, a queued dynamic gift is collected when no rematch battle is currently selected. This is the path relevant to item-only phone outcomes.

`ITEM_CHERI_BERRY` and `ITEM_POKE_BALL` are therefore not fixed dynamic phone gifts: either value is interpreted as a random-item sentinel before it is queued. A fixed Cheri Berry or fixed Poké Ball cannot be awarded through a normal `PHONESCRIPTTYPE_ITEM` definition. In contrast, the static rematch-reward path treats only Cheri Berry specially; a static `PhoneBookEntry.gift` value of `ITEM_POKE_BALL` remains a fixed Poké Ball.

:::caution
`GetPhoneContactGiftItem` clears a dynamic gift before the field script checks Bag space. If the Bag is full, the player receives the full-Bag message but the queued dynamic gift has already been lost.
:::

---

## Hex editing notes

<details>
<summary>Editing <code>pmtel_book.dat</code> directly</summary>

1. Extract or open the direct NitroFS file `tel/pmtel_book.dat`.
2. Read the 32-bit little-endian count at `0x00`.
3. Contact record `n` starts at `0x04 + (n * 0x14)`.
4. Use the table in [Phonebook data](#phonebook-data-pmtel_bookdat) for field offsets.
5. Keep the record count and record size unchanged unless you also understand every contact-ID-indexed table in the game.

For example, record-relative offset `0x0A` is the two-byte local generic outgoing script ID. Record-relative offset `0x0F` is the random-call bucket. All multi-byte values are little-endian.

Do not edit unknown fields, and do not add a new contact merely by appending a record: message archive mappings, persistent-state array indexes, contact constants, and other ID-indexed tables must also be expanded.

</details>

---

## See also

- [pret/pokeheartgold: phonebook reader and message mapping](https://github.com/pret/pokeheartgold/blob/master/src/phonebook_dat.c)
- [pret/pokeheartgold: phone contact structure](https://github.com/pret/pokeheartgold/blob/master/include/gear_phone.h)
- [pret/pokeheartgold: incoming call manager and event table](https://github.com/pret/pokeheartgold/blob/master/src/field/overlay_2_gear_phone.c)
- [pret/pokeheartgold: call direction/type dispatch](https://github.com/pret/pokeheartgold/blob/master/src/application/pokegear/phone/overlay_101_021F1D74.c)
- [pret/pokeheartgold: generic phone conditions](https://github.com/pret/pokeheartgold/blob/master/src/application/pokegear/phone/scripts/phone_scripts_generic.c)
- [pret/pokeheartgold: Elm handler](https://github.com/pret/pokeheartgold/blob/master/src/application/pokegear/phone/scripts/phone_scripts_prof_elm.c)
- [pret/pokeheartgold: Mom handler](https://github.com/pret/pokeheartgold/blob/master/src/application/pokegear/phone/scripts/phone_scripts_mother.c)
- [pret/pokeheartgold: call trigger constants](https://github.com/pret/pokeheartgold/blob/master/include/constants/phone_constants.h)
- [pret/pokeheartgold: map-header phone permissions](https://github.com/pret/pokeheartgold/blob/master/include/map_header.h)
- [pret/pokeheartgold: ordinary trainer rematches](https://github.com/pret/pokeheartgold/blob/master/src/overlay_26_022598C0.c)
- [DSPRE HGSS script command reference](https://github.com/DS-Pokemon-Rom-Editor/DSPRE/blob/master/DS_Map/Resources/HGSSCommands.md) - `0x0092` and `0x0093` use the legacy/editor command names shown above.
- [HGSS script command and ARM9 offset research](https://projectpokemon.org/home/forums/topic/54137-hgss-list-of-scripting-commands-and-arm9-offsets/)
- [Project Pokemon: saved Pokegear contact research](https://projectpokemon.org/home/forums/topic/49014-hgss-pokegear-phone-numbers-memory-location/)

<!-- PUBLISHING BLOCKER (Liam only): Submit an HGSS scrcmd-reference update before publishing. Define `0x0094` / `ScrCmd_148` as `SetPhoneCallTrigger triggerId, expedite` (two literal bytes; trigger IDs 0-12; nonzero expedite advances the shared incoming-call timer to threshold - 1), and define `0x0095` as `UnsetPhoneCallTrigger triggerId` (one literal byte; trigger IDs 0-12). -->
