---
description: Исходный код игрового смарт-контракта TON Fruits, написанный на языке FunC
---

# 💎 Код смарт-контракта TON Fruits

Также исходный код доступен [здесь](https://gist.github.com/ton-solutions/c300d0ebb0a3ee920c8e8b310a451e29)

```clike
global fee;
global ref_share;
;; Data
global int loaded;
global int seq_no;
global slice owner_address;
global int owner_public_key;
global int reels_count;
global int symbols_count;
global cell pay_table;
global cell affiliates;
global cell ref_codes;

() initialize_globals() impure {
  ifnot (null?(loaded)) {
    return ();
  }
  fee = 15; ;; (%)
  ref_share = 2; ;; (%)
  ;; Reading data
  ;; _ symbols_count:uint8 reels_count:uint8
  ;; pay_table:(HashmapE 32 uint32) = SlotParams
  ;;
  ;; _ seq_no:uint32 owner_address:MsgAddress owner_public_key:uint256
  ;; slot_params:^SlotParams affiliates:(HashmapE 32 MsgAddress)
  ;; ref_codes:(Hashmap 256 uint32) = Data
  var ds = get_data().begin_parse();
  seq_no = ds~load_uint(32);
  owner_address = ds~load_msg_addr();
  owner_public_key = ds~load_uint(256);
  var slot_params = ds~load_ref().begin_parse();
  symbols_count = slot_params~load_uint(8);
  reels_count = slot_params~load_uint(8);
  pay_table = slot_params~load_dict();
  affiliates = slice_empty?(ds) ? new_dict() : ds~load_dict();
  ref_codes = slice_empty?(ds) ? new_dict() : ds~load_dict();
  loaded = true;
}

slice get_owner() method_id {
  initialize_globals();
  return owner_address;
}

int get_seq_no() method_id {
  initialize_globals();
  return seq_no;
}

int get_max_stake_amount() method_id {
  initialize_globals();
  var (balance, _) = unpair(get_balance());
  var (_, max_combination, _) = udict_get_min?(pay_table, 32);
  var max_coefficient = max_combination~load_uint(32);
  return balance * 100 / (max_coefficient - 100);
}

int get_max_winning_amount() method_id {
  var (balance, _) = unpair(get_balance());
  return balance;
}

int get_gas_usage() method_id {
  return 37000;
}

cell get_gas_config_param() {
  var (wc, _) = parse_std_addr(my_address());
  ;;@ifnottest
  return config_param(21 + wc);
  ;;@iftest
  return begin_cell().store_uint(0xd1, 8).store_uint(0, 64).store_uint(0, 64).store_uint(0xde, 8).store_uint(0, 64).end_cell();
}

int get_min_stake_coefficient() method_id {
  return 8;
}

(int, int, int, int) get_state() method_id {
  initialize_globals();
  return (
    get_max_winning_amount(),
    get_max_stake_amount(),
    get_min_stake_coefficient(),
    get_gas_usage()
  );
}

;; gas_prices#dd gas_price:uint64 gas_limit:uint64 gas_credit:uint64 
;;   block_gas_limit:uint64 freeze_due_limit:uint64 delete_due_limit:uint64 
;;   = GasLimitsPrices;
;; 
;; gas_prices_ext#de gas_price:uint64 gas_limit:uint64 special_gas_limit:uint64 gas_credit:uint64 
;;   block_gas_limit:uint64 freeze_due_limit:uint64 delete_due_limit:uint64 
;;   = GasLimitsPrices;
;; 
;; gas_flat_pfx#d1 flat_gas_limit:uint64 flat_gas_price:uint64 other:GasLimitsPrices
;;   = GasLimitsPrices;
(slice, (int, int)) load_gas_flat_pfx(slice param) {
  var flat_gas_limit = param~load_uint(64);
  var flat_gas_price = param~load_uint(64);
  return (param, (flat_gas_limit, flat_gas_price));
}

(slice, int) load_gas_prices(slice param) {
  var gas_price = param~load_uint(64);
  return (param, gas_price);
}

(slice, (int, int, int)) load_gas_limits_prices(slice param) {
  var contructor_tag = param~load_uint(8);
  if (contructor_tag == 0xd1) {
    var (flat_gas_limit, flat_gas_price) = param~load_gas_flat_pfx();
    var (_, _, gas_price) = param~load_gas_limits_prices();
    return (param, (flat_gas_limit, flat_gas_price, gas_price));
  } elseif ((contructor_tag == 0xde) | (contructor_tag == 0xdd)) {
    var gas_price = param~load_gas_prices();
    return (param, (0, 0, gas_price));
  } else {
    throw_if(34, true);
    return (param, (0, 0, 0));
  }
}

(int, int, int) get_gas_limits_prices() {
  var gas_price_config = get_gas_config_param().begin_parse();
  return gas_price_config~load_gas_limits_prices();
}

int get_gas_fee() method_id {
  var (flat_gas_limit, flat_gas_price, gas_price) = get_gas_limits_prices();
  return get_gas_usage() < flat_gas_limit
    ? flat_gas_price
    : (get_gas_usage() - flat_gas_limit) * (gas_price >> 16) + flat_gas_price;
}

int get_min_stake_amount() method_id {
  return get_gas_fee() * get_min_stake_coefficient();
}

() store_data() impure {
  var slot_params = begin_cell()
    .store_uint(symbols_count, 8)
    .store_uint(reels_count, 8)
    .store_dict(pay_table)
    .end_cell();
  set_data(begin_cell()
    .store_uint(seq_no, 32)
    .store_slice(owner_address)
    .store_uint(owner_public_key, 256)
    .store_ref(slot_params)
    .store_dict(affiliates)
    .store_dict(ref_codes)
    .end_cell()
  );
}

() send_money(int wc, int addr, int amount, cell body, int mode) impure {
  var message = begin_cell()
    .store_uint(0, 1) ;; int_msg_info
      .store_uint(1, 1) ;; ihr_disabled
      .store_uint(0, 1) ;; bounce
      .store_uint(0, 1) ;; bounced
      .store_uint(0, 2) ;; src: addr_none
      .store_uint(2, 2) ;; dest: addr_std
        .store_uint(0, 1) ;; anycast
        .store_uint(wc, 8) ;; workchain_id
        .store_uint(addr, 256) ;; address
      .store_grams(amount) ;; grams
        .store_uint(0, 1) ;; other
      .store_grams(0) ;; ihr_fee
      .store_grams(0) ;; fwd_fee
      .store_uint(cur_lt(), 64) ;; created_lt
      .store_uint(now(), 32) ;; created_at
    .store_uint(0, 1) ;; init
    .store_uint(0, 1); ;; body
  ifnot (null?(body)) {
    message = message.store_slice(body.begin_parse());
  } else {
    message = message.store_uint(0, 32);
  }
  send_raw_message(message.end_cell(), null?(mode) ? 2 : mode); ;; ignore errors
}

(slice, int) parse_ref_code(slice comment) {
  if (slice_bits(comment) < 32) {
    return (comment, null());
  }
  var op = comment~load_uint(32);
  ifnot (op == 0) { ;; Not a comment
    return (comment, null());
  }
  int ref_code = null();
  if(comment~load_uint(8)) { ;; Dot character (.)
    ref_code = comment~load_uint(32);
  }
  return (comment, ref_code);
}

tuple insert_sorted(tuple t, int x) {
  if (null?(t)) {
    return cons(x, t);
  }
  var (y, tail) = uncons(t);
  if (x > y) {
    return cons(y, insert_sorted(tail, x));
  } else {
    return cons(x, t);
  }
}

;; Return codes
;; 32 - empty bid
;; 33 - stake is too small
;; 34 - unknown GasLimitsPrices constructor
;; 35 - empty comment
() recv_internal(int msg_value, cell in_msg_cell, slice in_msg) impure {
  ;; Throwing if bet is empty
  throw_if(32, msg_value == 0);
  var cs = in_msg_cell.begin_parse();
  cs~skip_bits(3);
  ;; When bounced do nothing
  var bounced = cs~load_uint(1);
  if (bounced == 1) {
    return ();
  }
  initialize_globals();
  ;; Reading player's address
  slice src_addr_slice = cs~load_msg_addr();
  var (src_wc, src_addr) = parse_std_addr(src_addr_slice);
  var (owner_wc, owner_addr) = parse_std_addr(owner_address);
  ;; When owner sends do nothing
  if ((owner_wc == src_wc) & (owner_addr == src_addr)) {
    return ();
  }
  ;; Throwing if stake is too small
  var gas_fee = get_gas_fee();
  throw_if(33, msg_value < get_min_stake_coefficient() * gas_fee);
  ;; Substracting gas fee
  var stake = msg_value - gas_fee;
  ;; Reading message payload
  int ref_code = null();
  var (ref_code_slice, _) = ref_codes.udict_get?(256, src_addr);
  ifnot (null?(ref_code_slice)) {
    ref_code = ref_code_slice~load_uint(32);
  }
  if (null?(ref_code)) {
    ref_code = in_msg~parse_ref_code();
    ifnot (null?(ref_code)) {
      ref_codes = ref_codes.udict_set(256, src_addr, begin_cell().store_uint(ref_code, 32).end_cell().begin_parse());
      store_data();
      commit();
    }
  }
  throw_if(35, null?(ref_code));
  ;; Calculating change
  var change = max(0, stake - get_max_stake_amount());
  stake -= change;
  ;; Making a spin
  var reels = null();
  repeat (reels_count) {
    reels = cons(rand(symbols_count), reels);
    randomize_lt();
  }
  ;; Counting repeating symbols and calculating winning_multiplier
  var current_reel = reels;
  int counted_symbols = 0; ;; Set of already counted symbols
  var total_occurrencies = null();
  while(~(null?(current_reel))) {
    int symbol = current_reel~list_next();
    var shift = 1 << symbol;
    if ((shift & counted_symbols) != shift) { ;; Skipping if symbol already counted
      counted_symbols |= shift; ;; Adding symbol to the set
      var tail = current_reel;
      int occurrencies = 1;
      while(~(null?(tail))) {
        int next = tail~list_next();
        if (next == symbol) {
          occurrencies += 1;
        }
      }
      total_occurrencies = insert_sorted(total_occurrencies, occurrencies);
    }
  }
  var combination = 0;
  var mult = 1;
  while(~(null?(total_occurrencies))) {
    combination += (total_occurrencies~list_next() * mult);
    mult *= 10;
  }
  var winning_multiplier = 0;
  var (coef_slice, _) = pay_table.udict_get?(32, combination);
  ifnot (null?(coef_slice)) {
    winning_multiplier = coef_slice~load_uint(32);
  }
  ;; Calculating winning
  var winning = winning_multiplier * stake / 100;
  ;; Creating spinlog message body
  var reels_slice = begin_cell();
  repeat(reels_count) {
    reels_slice = reels_slice.store_uint(reels~list_next(), 8);
  }
  var reels_cell = reels_slice.end_cell();
  var body = begin_cell()
    .store_uint(0xffffffff, 32) ;; op
    .store_grams(stake)
    .store_grams(winning)
    .store_grams(change)
    .store_grams(gas_fee)
    .store_slice(reels_cell.begin_parse())
    .end_cell();
  ;; Sending winning + change
  var amount_to_send_back = change + winning;
  if (amount_to_send_back > 0) {
    send_money(src_wc, src_addr, amount_to_send_back, body, null());
    commit();
  }
  ;; Sending fee
  var income = max(0, stake - winning);
  if (income > 0) {
    var revenue = income * fee / 100;
    send_money(owner_wc, owner_addr, revenue, amount_to_send_back > 0 ? null() : body, null());
    commit();
    var reminder = income - revenue;
    ifnot (null?(ref_code)) {
      var (affiliate, _) = affiliates.udict_get?(32, ref_code);
      ifnot (null?(affiliate)) {
        var (aff_wc, aff_addr) = parse_std_addr(affiliate);
        var share = min(reminder, income * ref_share / 100);
        var body = begin_cell()
          .store_uint(0, 32)
          .store_uint(0x68747470733a2f2f742e6d652f746f6e5f6672756974735f626f74, 27 * 8) ;; https://t.me/ton_fruits_bot
          .store_uint(0x207061796f7574, 7 * 8) ;; payout
          .end_cell();
        send_money(aff_wc, aff_addr, share, body, null());
        commit();
      }
    }
  }
}

;; op 0 - Inital deploy
;; op 1 - Update code
;; op 2 - Add affiliate
;; op 3 - Set owner
;; op 666 - Self destruct
;; Return codes
;; 32 - Invalid signature
;; 33 - Invalid seq_no
() recv_external(slice in_msg) impure {
  initialize_globals();
  var signature = in_msg~load_bits(512);
  var hash = slice_hash(in_msg);
  throw_unless(32, check_signature(hash, signature, owner_public_key));
  var msg_seq_no = in_msg~load_uint(32);
  throw_if(33, msg_seq_no != seq_no);
  if (seq_no == 0) {
    accept_message();
  } else {
    var op = in_msg~load_uint(32);
    ;; Update code
    if (op == 1) {
      accept_message();
      var code_cell = in_msg~load_ref();
      set_code(code_cell);
    }
    ;; Add affiliate
    elseif (op == 2) {
      accept_message();
      var ref_code = in_msg~load_uint(32);
      var address = in_msg~load_msg_addr();
      affiliates = affiliates.udict_set(32, ref_code, address);
    }
    ;; Set owner
    elseif (op == 3) {
      accept_message();
      owner_address = in_msg~load_msg_addr();
    }
    ;; Self destruct
    elseif (op == 666) {
      accept_message();
      var (owner_wc, owner_addr) = parse_std_addr(owner_address);
      send_money(owner_wc, owner_addr, 0, null(), 128 + 32);
    }
  }
  seq_no += 1;
  store_data();
f
```
