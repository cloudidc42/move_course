# Part 18: Strings Deep Dive

## สารบัญ
- [String Types ใน Move](#string-types-ใน-move)
- [String Operations](#string-operations)
- [UTF-8 Handling](#utf-8-handling)
- [Number Conversion](#number-conversion)
- [String Patterns](#string-patterns)
- [ตัวอย่างโปรแกรม: Token Metadata Engine](#ตัวอย่างโปรแกรม-token-metadata-engine)

---

## String Types ใน Move

```move
module learning::string_types {
    use std::string::{Self, String};
    
    // ============================================
    // Two string types
    // ============================================
    
    // 1. vector<u8> - raw bytes
    //    - Lower level
    //    - No UTF-8 validation
    //    - Used for binary data
    
    // 2. std::string::String - UTF-8 string
    //    - Higher level  
    //    - UTF-8 validated on creation
    //    - Used for human-readable text
    
    public fun string_types_demo() {
        // Raw bytes (no UTF-8 validation)
        let raw_bytes: vector<u8> = b"Hello, World!";
        
        // UTF-8 String (validates on creation)
        let s: String = string::utf8(b"Hello, World!");
        
        // Convert String to bytes
        let bytes: &vector<u8> = string::bytes(&s);
        
        // From vector<u8>
        let s2: String = string::utf8(raw_bytes);
        
        let _ = (s, bytes, s2);
    }
    
    // ============================================
    // String vs vector<u8> when to use each
    // ============================================
    
    // Use vector<u8> for:
    // - Hashes, signatures, binary data
    // - Low-level byte manipulation
    // - Protocol data
    
    struct MessageHash {
        hash: vector<u8>,     // 32 bytes hash
        signature: vector<u8>, // raw signature
    }
    
    // Use String for:
    // - Names, descriptions, URIs
    // - Human-readable metadata
    // - Any text users will see
    
    struct TokenMetadata {
        name: String,
        symbol: String,
        description: String,
        uri: String,
    }
    
    // ============================================
    // string::String abilities
    // ============================================
    
    // String has: copy, drop, store
    // So you can:
    
    public fun string_ability_demo() {
        let s = string::utf8(b"hello");
        let s2 = s;   // copy
        let s3 = s;   // copy again
        // All three can be used
        let _ = (s2, s3);
        // s is automatically dropped
    }
}
```

---

## String Operations

```move
module learning::string_ops {
    use std::string::{Self, String};
    use std::vector;
    
    // ============================================
    // Building strings
    // ============================================
    
    public fun concat(a: String, b: String): String {
        let mut_a = a;
        string::append(&mut mut_a, b);
        mut_a
    }
    
    public fun concat_bytes(s: String, bytes: vector<u8>): String {
        let mut_s = s;
        string::append_utf8(&mut mut_s, bytes);
        mut_s
    }
    
    public fun join(parts: &vector<String>, separator: String): String {
        let result = string::utf8(b"");
        let len = vector::length(parts);
        let i = 0u64;
        while (i < len) {
            if (i > 0) {
                string::append(&mut result, separator);
            };
            string::append(&mut result, *vector::borrow(parts, i));
            i = i + 1;
        };
        result
    }
    
    // ============================================
    // String inspection
    // ============================================
    
    public fun starts_with(s: &String, prefix: &String): bool {
        let s_bytes = string::bytes(s);
        let p_bytes = string::bytes(prefix);
        
        if (vector::length(s_bytes) < vector::length(p_bytes)) {
            return false
        };
        
        let i = 0u64;
        while (i < vector::length(p_bytes)) {
            if (*vector::borrow(s_bytes, i) != *vector::borrow(p_bytes, i)) {
                return false
            };
            i = i + 1;
        };
        true
    }
    
    public fun ends_with(s: &String, suffix: &String): bool {
        let s_bytes = string::bytes(s);
        let suf_bytes = string::bytes(suffix);
        
        let s_len = vector::length(s_bytes);
        let suf_len = vector::length(suf_bytes);
        
        if (s_len < suf_len) return false;
        
        let offset = s_len - suf_len;
        let i = 0u64;
        while (i < suf_len) {
            if (*vector::borrow(s_bytes, offset + i) != *vector::borrow(suf_bytes, i)) {
                return false
            };
            i = i + 1;
        };
        true
    }
    
    public fun contains_str(s: &String, sub: &String): bool {
        let s_bytes = string::bytes(s);
        let sub_bytes = string::bytes(sub);
        
        let s_len = vector::length(s_bytes);
        let sub_len = vector::length(sub_bytes);
        
        if (s_len < sub_len) return false;
        
        let i = 0u64;
        while (i <= s_len - sub_len) {
            let j = 0u64;
            let matched = true;
            while (j < sub_len) {
                if (*vector::borrow(s_bytes, i + j) != *vector::borrow(sub_bytes, j)) {
                    matched = false;
                    break
                };
                j = j + 1;
            };
            if (matched) return true;
            i = i + 1;
        };
        false
    }
    
    // ============================================
    // Splitting and trimming
    // ============================================
    
    public fun trim_leading_spaces(s: &String): String {
        let bytes = string::bytes(s);
        let len = vector::length(bytes);
        let start = 0u64;
        
        while (start < len && *vector::borrow(bytes, start) == 32u8) {  // 32 = space
            start = start + 1;
        };
        
        let result_bytes = vector::empty<u8>();
        let i = start;
        while (i < len) {
            vector::push_back(&mut result_bytes, *vector::borrow(bytes, i));
            i = i + 1;
        };
        string::utf8(result_bytes)
    }
    
    public fun split_by_char(s: &String, delimiter: u8): vector<String> {
        let bytes = string::bytes(s);
        let len = vector::length(bytes);
        let result = vector::empty<String>();
        let current = vector::empty<u8>();
        
        let i = 0u64;
        while (i < len) {
            let c = *vector::borrow(bytes, i);
            if (c == delimiter) {
                vector::push_back(&mut result, string::utf8(current));
                current = vector::empty<u8>();
            } else {
                vector::push_back(&mut current, c);
            };
            i = i + 1;
        };
        
        // Add last part
        vector::push_back(&mut result, string::utf8(current));
        result
    }
    
    // ============================================
    // Case conversion
    // ============================================
    
    public fun to_uppercase(s: &String): String {
        let bytes = string::bytes(s);
        let result = vector::empty<u8>();
        let i = 0u64;
        while (i < vector::length(bytes)) {
            let c = *vector::borrow(bytes, i);
            // a-z = 97-122, A-Z = 65-90
            let upper = if (c >= 97 && c <= 122) {
                c - 32
            } else {
                c
            };
            vector::push_back(&mut result, upper);
            i = i + 1;
        };
        string::utf8(result)
    }
    
    public fun to_lowercase(s: &String): String {
        let bytes = string::bytes(s);
        let result = vector::empty<u8>();
        let i = 0u64;
        while (i < vector::length(bytes)) {
            let c = *vector::borrow(bytes, i);
            let lower = if (c >= 65 && c <= 90) {
                c + 32
            } else {
                c
            };
            vector::push_back(&mut result, lower);
            i = i + 1;
        };
        string::utf8(result)
    }
    
    // ============================================
    // Tests
    // ============================================
    
    #[test]
    public fun test_string_ops() {
        let s = string::utf8(b"Hello, World!");
        let prefix = string::utf8(b"Hello");
        let suffix = string::utf8(b"World!");
        
        assert!(starts_with(&s, &prefix), 0);
        assert!(ends_with(&s, &suffix), 1);
        assert!(contains_str(&s, &string::utf8(b"World")), 2);
        assert!(!contains_str(&s, &string::utf8(b"xyz")), 3);
    }
    
    #[test]
    public fun test_split() {
        let s = string::utf8(b"a,b,c,d");
        let parts = split_by_char(&s, 44u8);  // 44 = ','
        assert!(vector::length(&parts) == 4, 0);
        assert!(*vector::borrow(&parts, 0) == string::utf8(b"a"), 1);
        assert!(*vector::borrow(&parts, 3) == string::utf8(b"d"), 2);
    }
    
    #[test]
    public fun test_case() {
        let s = string::utf8(b"Hello World");
        let upper = to_uppercase(&s);
        let lower = to_lowercase(&s);
        
        assert!(upper == string::utf8(b"HELLO WORLD"), 0);
        assert!(lower == string::utf8(b"hello world"), 1);
    }
}
```

---

## Number Conversion

```move
module learning::number_conversion {
    use std::string::{Self, String};
    use std::vector;
    
    // ============================================
    // Integer to String
    // ============================================
    
    public fun u64_to_string(n: u64): String {
        if (n == 0) {
            return string::utf8(b"0")
        };
        
        let bytes = vector::empty<u8>();
        let n_mut = n;
        while (n_mut > 0) {
            let digit = ((n_mut % 10) as u8) + 48;  // '0' = 48
            vector::push_back(&mut bytes, digit);
            n_mut = n_mut / 10;
        };
        
        vector::reverse(&mut bytes);
        string::utf8(bytes)
    }
    
    public fun u128_to_string(n: u128): String {
        if (n == 0) {
            return string::utf8(b"0")
        };
        
        let bytes = vector::empty<u8>();
        let n_mut = n;
        while (n_mut > 0) {
            let digit = ((n_mut % 10) as u8) + 48;
            vector::push_back(&mut bytes, digit);
            n_mut = n_mut / 10;
        };
        
        vector::reverse(&mut bytes);
        string::utf8(bytes)
    }
    
    // Format with decimal places
    // E.g., 1000000000 with decimals=9 → "1.000000000"
    public fun format_with_decimals(amount: u64, decimals: u8): String {
        if (decimals == 0) {
            return u64_to_string(amount)
        };
        
        let divisor = pow10(decimals);
        let integer_part = amount / divisor;
        let fractional_part = amount % divisor;
        
        let result = u64_to_string(integer_part);
        string::append_utf8(&mut result, b".");
        
        // Pad with leading zeros
        let frac_str = u64_to_string(fractional_part);
        let frac_len = string::length(&frac_str);
        let needed_zeros = (decimals as u64) - frac_len;
        
        let i = 0u64;
        while (i < needed_zeros) {
            string::append_utf8(&mut result, b"0");
            i = i + 1;
        };
        
        string::append(&mut result, frac_str);
        result
    }
    
    fun pow10(exp: u8): u64 {
        let result = 1u64;
        let i = 0u8;
        while (i < exp) {
            result = result * 10;
            i = i + 1;
        };
        result
    }
    
    // Format as percentage with 2 decimal places
    // E.g., bps=1234 → "12.34%"
    public fun format_bps_as_percentage(bps: u64): String {
        let integer = bps / 100;
        let fractional = bps % 100;
        
        let result = u64_to_string(integer);
        string::append_utf8(&mut result, b".");
        
        if (fractional < 10) {
            string::append_utf8(&mut result, b"0");
        };
        string::append(&mut result, u64_to_string(fractional));
        string::append_utf8(&mut result, b"%");
        result
    }
    
    // ============================================
    // String to Integer (parsing)
    // ============================================
    
    public fun parse_u64(s: &String): (bool, u64) {
        let bytes = string::bytes(s);
        let len = vector::length(bytes);
        
        if (len == 0) return (false, 0);
        
        let result = 0u64;
        let i = 0u64;
        
        while (i < len) {
            let c = *vector::borrow(bytes, i);
            if (c < 48 || c > 57) {  // not '0'-'9'
                return (false, 0)
            };
            
            let digit = (c - 48) as u64;
            
            // Check overflow
            if (result > (18446744073709551615u64 - digit) / 10) {
                return (false, 0)
            };
            
            result = result * 10 + digit;
            i = i + 1;
        };
        
        (true, result)
    }
    
    // ============================================
    // Hex encoding/decoding
    // ============================================
    
    public fun bytes_to_hex(bytes: &vector<u8>): String {
        let hex_chars = b"0123456789abcdef";
        let result = vector::empty<u8>();
        
        let i = 0u64;
        while (i < vector::length(bytes)) {
            let byte = *vector::borrow(bytes, i);
            let high = byte >> 4;
            let low = byte & 0xF;
            
            vector::push_back(&mut result, *vector::borrow(&hex_chars, (high as u64)));
            vector::push_back(&mut result, *vector::borrow(&hex_chars, (low as u64)));
            
            i = i + 1;
        };
        
        string::utf8(result)
    }
    
    public fun address_to_string(addr: address): String {
        // Convert address to hex string
        let bytes = std::bcs::to_bytes(&addr);
        let hex = bytes_to_hex(&bytes);
        let mut_str = string::utf8(b"0x");
        string::append(&mut mut_str, hex);
        mut_str
    }
    
    // ============================================
    // Tests
    // ============================================
    
    #[test]
    public fun test_number_conversion() {
        assert!(u64_to_string(0) == string::utf8(b"0"), 0);
        assert!(u64_to_string(42) == string::utf8(b"42"), 1);
        assert!(u64_to_string(1000000) == string::utf8(b"1000000"), 2);
        assert!(u64_to_string(18446744073709551615u64) == string::utf8(b"18446744073709551615"), 3);
    }
    
    #[test]
    public fun test_format_with_decimals() {
        // 1 APT = 100000000 (8 decimals)
        let result = format_with_decimals(150000000, 8);
        assert!(result == string::utf8(b"1.50000000"), 0);
        
        let result2 = format_with_decimals(1000000000, 8);
        assert!(result2 == string::utf8(b"10.00000000"), 1);
    }
    
    #[test]
    public fun test_parse_u64() {
        let (ok1, v1) = parse_u64(&string::utf8(b"42"));
        assert!(ok1 && v1 == 42, 0);
        
        let (ok2, v2) = parse_u64(&string::utf8(b"0"));
        assert!(ok2 && v2 == 0, 1);
        
        let (ok3, _) = parse_u64(&string::utf8(b"abc"));
        assert!(!ok3, 2);
    }
    
    #[test]
    public fun test_percentage_format() {
        let p = format_bps_as_percentage(1234);
        assert!(p == string::utf8(b"12.34%"), 0);
        
        let p2 = format_bps_as_percentage(500);
        assert!(p2 == string::utf8(b"5.00%"), 1);
        
        let p3 = format_bps_as_percentage(10000);
        assert!(p3 == string::utf8(b"100.00%"), 2);
    }
}
```

---

## String Patterns

```move
module learning::string_patterns {
    use std::string::{Self, String};
    use std::vector;
    
    // ============================================
    // URI building
    // ============================================
    
    public fun build_token_uri(
        base_uri: String,
        token_id: u64,
    ): String {
        let uri = base_uri;
        string::append_utf8(&mut uri, b"/");
        
        // Append token ID
        let id_str = super::number_conversion::u64_to_string(token_id);
        string::append(&mut uri, id_str);
        
        string::append_utf8(&mut uri, b".json");
        uri
    }
    
    // ============================================
    // Template system
    // ============================================
    
    // Simple template: replace {0}, {1}, etc.
    public fun format_template(template: String, args: &vector<String>): String {
        let result = string::utf8(b"");
        let bytes = string::bytes(&template);
        let len = vector::length(bytes);
        let i = 0u64;
        
        while (i < len) {
            let c = *vector::borrow(bytes, i);
            
            // Look for {N} patterns
            if (c == 123u8 && i + 2 < len) {  // '{'
                let next = *vector::borrow(bytes, i + 1);
                let close = *vector::borrow(bytes, i + 2);
                
                if (next >= 48 && next <= 57 && close == 125u8) {  // digit + '}'
                    let arg_idx = (next - 48) as u64;
                    if (arg_idx < vector::length(args)) {
                        string::append(&mut result, *vector::borrow(args, arg_idx));
                        i = i + 3;
                        continue
                    };
                };
            };
            
            let char_bytes = vector[c];
            string::append_utf8(&mut result, char_bytes);
            i = i + 1;
        };
        
        result
    }
    
    // ============================================
    // JSON-like string building
    // ============================================
    
    public fun build_json_object(
        keys: &vector<String>,
        values: &vector<String>,
    ): String {
        let len = vector::length(keys);
        assert!(len == vector::length(values), 0);
        
        let result = string::utf8(b"{");
        let i = 0u64;
        while (i < len) {
            if (i > 0) {
                string::append_utf8(&mut result, b",");
            };
            string::append_utf8(&mut result, b"\"");
            string::append(&mut result, *vector::borrow(keys, i));
            string::append_utf8(&mut result, b"\":\"");
            string::append(&mut result, *vector::borrow(values, i));
            string::append_utf8(&mut result, b"\"");
            i = i + 1;
        };
        string::append_utf8(&mut result, b"}");
        result
    }
    
    // ============================================
    // NFT Metadata JSON
    // ============================================
    
    public fun build_nft_metadata(
        name: String,
        description: String,
        image_uri: String,
        trait_types: &vector<String>,
        trait_values: &vector<String>,
    ): String {
        let mut_result = string::utf8(b"{\"name\":\"");
        string::append(&mut mut_result, name);
        string::append_utf8(&mut mut_result, b"\",\"description\":\"");
        string::append(&mut mut_result, description);
        string::append_utf8(&mut mut_result, b"\",\"image\":\"");
        string::append(&mut mut_result, image_uri);
        string::append_utf8(&mut mut_result, b"\",\"attributes\":[");
        
        let len = vector::length(trait_types);
        let i = 0u64;
        while (i < len) {
            if (i > 0) {
                string::append_utf8(&mut mut_result, b",");
            };
            string::append_utf8(&mut mut_result, b"{\"trait_type\":\"");
            string::append(&mut mut_result, *vector::borrow(trait_types, i));
            string::append_utf8(&mut mut_result, b"\",\"value\":\"");
            string::append(&mut mut_result, *vector::borrow(trait_values, i));
            string::append_utf8(&mut mut_result, b"\"}");
            i = i + 1;
        };
        
        string::append_utf8(&mut mut_result, b"]}");
        mut_result
    }
    
    // ============================================
    // Tests
    // ============================================
    
    #[test]
    public fun test_build_json() {
        let keys = vector[
            string::utf8(b"name"),
            string::utf8(b"value"),
        ];
        let values = vector[
            string::utf8(b"Alice"),
            string::utf8(b"100"),
        ];
        
        let json = build_json_object(&keys, &values);
        assert!(json == string::utf8(b"{\"name\":\"Alice\",\"value\":\"100\"}"), 0);
    }
    
    #[test]
    public fun test_nft_metadata() {
        let traits = vector[
            string::utf8(b"Background"),
            string::utf8(b"Rarity"),
        ];
        let trait_vals = vector[
            string::utf8(b"Blue"),
            string::utf8(b"Legendary"),
        ];
        
        let metadata = build_nft_metadata(
            string::utf8(b"Epic Sword"),
            string::utf8(b"A legendary weapon"),
            string::utf8(b"https://example.com/1.png"),
            &traits,
            &trait_vals,
        );
        
        // Check it contains expected parts
        use learning::string_ops;
        assert!(string_ops::contains_str(&metadata, &string::utf8(b"Epic Sword")), 0);
        assert!(string_ops::contains_str(&metadata, &string::utf8(b"Legendary")), 1);
    }
}
```

---

## ตัวอย่างโปรแกรม: Token Metadata Engine

```move
module learning::metadata_engine {
    use std::signer;
    use std::string::{Self, String};
    use std::vector;
    use aptos_std::table::{Self, Table};
    
    // ============================================
    // Metadata Types
    // ============================================
    
    struct Attribute has copy, drop, store {
        trait_type: String,
        value: String,
        display_type: String,  // "string", "number", "date", "boost_percentage"
    }
    
    struct TokenMetadata has key, store {
        name: String,
        symbol: String,
        decimals: u8,
        description: String,
        icon_uri: String,
        project_uri: String,
        attributes: vector<Attribute>,
        metadata_uri: String,
        version: u64,
    }
    
    struct MetadataRegistry has key {
        entries: Table<address, TokenMetadata>,
        count: u64,
    }
    
    // ============================================
    // Initialization
    // ============================================
    
    public entry fun initialize_registry(admin: &signer) {
        move_to(admin, MetadataRegistry {
            entries: table::new(),
            count: 0,
        });
    }
    
    // ============================================
    // Register Token
    // ============================================
    
    public entry fun register_token(
        creator: &signer,
        registry_addr: address,
        token_addr: address,
        name: vector<u8>,
        symbol: vector<u8>,
        decimals: u8,
        description: vector<u8>,
        icon_uri: vector<u8>,
        project_uri: vector<u8>,
    ) acquires MetadataRegistry {
        let registry = borrow_global_mut<MetadataRegistry>(registry_addr);
        assert!(!table::contains(&registry.entries, token_addr), 1);
        
        table::add(&mut registry.entries, token_addr, TokenMetadata {
            name: string::utf8(name),
            symbol: string::utf8(symbol),
            decimals,
            description: string::utf8(description),
            icon_uri: string::utf8(icon_uri),
            project_uri: string::utf8(project_uri),
            attributes: vector::empty(),
            metadata_uri: generate_metadata_uri(token_addr),
            version: 1,
        });
        
        registry.count = registry.count + 1;
    }
    
    // ============================================
    // Add Attributes
    // ============================================
    
    public entry fun add_attribute(
        creator: &signer,
        registry_addr: address,
        token_addr: address,
        trait_type: vector<u8>,
        value: vector<u8>,
        display_type: vector<u8>,
    ) acquires MetadataRegistry {
        let registry = borrow_global_mut<MetadataRegistry>(registry_addr);
        let metadata = table::borrow_mut(&mut registry.entries, token_addr);
        
        vector::push_back(&mut metadata.attributes, Attribute {
            trait_type: string::utf8(trait_type),
            value: string::utf8(value),
            display_type: string::utf8(display_type),
        });
        
        metadata.version = metadata.version + 1;
    }
    
    // ============================================
    // Generate Standard JSON Metadata
    // ============================================
    
    public fun generate_json_metadata(
        registry_addr: address,
        token_addr: address,
    ): String acquires MetadataRegistry {
        let registry = borrow_global<MetadataRegistry>(registry_addr);
        let m = table::borrow(&registry.entries, token_addr);
        
        let result = string::utf8(b"{");
        
        // Basic fields
        append_json_string(&mut result, b"name", string::bytes(&m.name), false);
        append_json_string(&mut result, b"symbol", string::bytes(&m.symbol), true);
        
        // Decimals as number
        string::append_utf8(&mut result, b",\"decimals\":");
        let dec_str = num_to_string((m.decimals as u64));
        string::append(&mut result, dec_str);
        
        // Description
        append_json_string(&mut result, b"description", string::bytes(&m.description), true);
        
        // URIs
        append_json_string(&mut result, b"icon_uri", string::bytes(&m.icon_uri), true);
        append_json_string(&mut result, b"project_uri", string::bytes(&m.project_uri), true);
        
        // Attributes array
        string::append_utf8(&mut result, b",\"attributes\":[");
        let i = 0u64;
        while (i < vector::length(&m.attributes)) {
            if (i > 0) string::append_utf8(&mut result, b",");
            let attr = vector::borrow(&m.attributes, i);
            
            string::append_utf8(&mut result, b"{");
            append_json_string(&mut result, b"trait_type", string::bytes(&attr.trait_type), false);
            append_json_string(&mut result, b"value", string::bytes(&attr.value), true);
            if (!string::is_empty(&attr.display_type)) {
                append_json_string(&mut result, b"display_type", string::bytes(&attr.display_type), true);
            };
            string::append_utf8(&mut result, b"}");
            i = i + 1;
        };
        string::append_utf8(&mut result, b"]");
        
        string::append_utf8(&mut result, b"}");
        result
    }
    
    // ============================================
    // Helpers
    // ============================================
    
    fun append_json_string(
        result: &mut String,
        key: vector<u8>,
        value: &vector<u8>,
        comma_prefix: bool,
    ) {
        if (comma_prefix) {
            string::append_utf8(result, b",");
        };
        string::append_utf8(result, b"\"");
        string::append_utf8(result, key);
        string::append_utf8(result, b"\":\"");
        string::append_utf8(result, *value);
        string::append_utf8(result, b"\"");
    }
    
    fun generate_metadata_uri(token_addr: address): String {
        let uri = string::utf8(b"https://metadata.example.com/");
        let addr_bytes = std::bcs::to_bytes(&token_addr);
        // Append hex-encoded address
        let hex = address_to_short_hex(&addr_bytes);
        string::append(&mut uri, hex);
        uri
    }
    
    fun address_to_short_hex(bytes: &vector<u8>): String {
        let hex_chars = b"0123456789abcdef";
        let result = vector::empty<u8>();
        let len = if (vector::length(bytes) > 8) { 8 } else { vector::length(bytes) };
        let i = 0u64;
        while (i < len) {
            let byte = *vector::borrow(bytes, i);
            let high = byte >> 4;
            let low = byte & 0xF;
            vector::push_back(&mut result, *vector::borrow(&hex_chars, (high as u64)));
            vector::push_back(&mut result, *vector::borrow(&hex_chars, (low as u64)));
            i = i + 1;
        };
        string::utf8(result)
    }
    
    fun num_to_string(n: u64): String {
        if (n == 0) return string::utf8(b"0");
        let bytes = vector::empty<u8>();
        let n_mut = n;
        while (n_mut > 0) {
            vector::push_back(&mut bytes, ((n_mut % 10) as u8) + 48);
            n_mut = n_mut / 10;
        };
        vector::reverse(&mut bytes);
        string::utf8(bytes)
    }
    
    // ============================================
    // View Functions
    // ============================================
    
    #[view]
    public fun get_token_name(
        registry_addr: address,
        token_addr: address,
    ): String acquires MetadataRegistry {
        let registry = borrow_global<MetadataRegistry>(registry_addr);
        table::borrow(&registry.entries, token_addr).name
    }
    
    #[view]
    public fun get_attribute_count(
        registry_addr: address,
        token_addr: address,
    ): u64 acquires MetadataRegistry {
        let registry = borrow_global<MetadataRegistry>(registry_addr);
        vector::length(&table::borrow(&registry.entries, token_addr).attributes)
    }
    
    // ============================================
    // Tests
    // ============================================
    
    #[test(admin = @0x1, creator = @0x2)]
    public fun test_metadata_engine(admin: &signer, creator: &signer) acquires MetadataRegistry {
        let admin_addr = signer::address_of(admin);
        let token_addr = @0x100;
        
        initialize_registry(admin);
        
        register_token(
            creator, admin_addr, token_addr,
            b"MyToken", b"MTK", 8,
            b"A test token", b"https://icon.example.com", b"https://mytoken.example.com",
        );
        
        add_attribute(creator, admin_addr, token_addr, b"Category", b"DeFi", b"string");
        add_attribute(creator, admin_addr, token_addr, b"Max Supply", b"1000000", b"number");
        
        assert!(get_token_name(admin_addr, token_addr) == string::utf8(b"MyToken"), 0);
        assert!(get_attribute_count(admin_addr, token_addr) == 2, 1);
        
        let json = generate_json_metadata(admin_addr, token_addr);
        use learning::string_ops;
        assert!(string_ops::contains_str(&json, &string::utf8(b"MyToken")), 2);
        assert!(string_ops::contains_str(&json, &string::utf8(b"DeFi")), 3);
    }
}
```

---

## สรุป Strings

| Operation | Function |
|-----------|---------|
| Create | `string::utf8(bytes)` |
| Append | `string::append(&mut s, other)` |
| Append bytes | `string::append_utf8(&mut s, b"...")` |
| Length | `string::length(&s)` |
| Empty check | `string::is_empty(&s)` |
| Get bytes | `string::bytes(&s)` |
| Equality | `s1 == s2` |

---

**ต่อไป**: [Part 19 - Math and Fixed-Point →](part-19-math.md)
