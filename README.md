# AVBTOOL Mod

English | [中文](README_zh.md)

Original project: https://github.com/AndroidBootloader/platform_external_avb

AVBTOOL Mod is a standalone version of `avbtool` extracted from the original Android Verified Boot project. It keeps the upstream command surface while running RSA, signature checks, and bundled FEC in-process.

## Added changes

The changes made by AVBTOOL Mod:

- `--pass-file <FILE>`: loads the passphrase for an encrypted RSA key in-process on signing commands, `extract_public_key`, and `verify_image`. The key is not written to a decrypted temporary file.
- `--dynamic_partition_size`: automatically calculates the partition size for `add_hash_footer` and `add_hashtree_footer`. It cannot be combined with `--partition_size` or `--calc_max_image_size`.
- Bundled FEC: `add_hashtree_footer` generates AOSP-compatible FEC data by default, including its 4 KiB footer, without requiring the external `fec` tool.
- In-process cryptography: PEM and DER RSA keys are parsed and signed with `cryptography`. `verify_image` checks signatures without invoking OpenSSL.
- Python 3 output fixes: commands that write binary data can use stdout without the binary-stream `AttributeError`.

## Requirements

- Python 3.11 or newer
- the `cryptography` package

```sh
python3 -m pip install cryptography
python3 avbtool.py --help
```

The manual build workflow produces Ubuntu 22.04 amd64 and arm64 executables with both PyInstaller and Nuitka. Each artifact is named `avbtool-mod-<version>-<arch>-<builder>-<Shanghai date>.zip` and contains `avbtool`; both builders bundle `cryptography`.

## Command tree

The following tree is generated from the current `argparse` command definitions in `avbtool.py`. Commands marked with an asterisk also accept the shared option groups below.

```text
avbtool
├── generate_test_image
│   ├── --image_size <NUMBER>              required
│   ├── --start_byte <NUMBER>              default: 0
│   └── --output <FILE>                    default: stdout
├── version
├── extract_public_key
│   ├── --key <FILE>                       required
│   ├── --output <FILE>                    required
│   └── --pass-file <FILE>
├── make_vbmeta_image *
│   ├── --output <FILE>
│   └── --padding_size <NUMBER>            default: 0
├── add_hash_footer *
│   ├── --image <FILE>
│   ├── --partition_size <NUMBER>          default: 0
│   ├── --dynamic_partition_size
│   ├── --partition_name <NAME>
│   ├── --hash_algorithm <NAME>            default: sha256
│   ├── --salt <HEX>                       default: random
│   ├── --calc_max_image_size
│   ├── --output_vbmeta_image <FILE>
│   ├── --do_not_append_vbmeta_image
│   └── Footer options
├── append_vbmeta_image
│   ├── --image <FILE>                     required
│   ├── --partition_size <NUMBER>          required
│   └── --vbmeta_image <FILE>              required
├── add_hashtree_footer *
│   ├── --image <FILE>
│   ├── --partition_size <NUMBER>          default: 0
│   ├── --dynamic_partition_size
│   ├── --partition_name <NAME>            default: empty
│   ├── --hash_algorithm <NAME>            default: sha1
│   ├── --salt <HEX>                       default: random
│   ├── --block_size <NUMBER>              default: 4096
│   ├── --do_not_generate_fec
│   ├── --fec_num_roots <NUMBER>           default: 2
│   ├── --calc_max_image_size
│   ├── --output_vbmeta_image <FILE>
│   ├── --do_not_append_vbmeta_image
│   ├── --setup_as_rootfs_from_kernel
│   ├── --no_hashtree
│   └── Footer options
├── erase_footer
│   ├── --image <FILE>                     required
│   └── --keep_hashtree
├── zero_hashtree
│   └── --image <FILE>                     required
├── extract_vbmeta_image
│   ├── --image <FILE>                     required
│   ├── --output <FILE>                    default: stdout
│   └── --padding_size <NUMBER>            default: 0
├── resize_image
│   ├── --image <FILE>                     required
│   └── --partition_size <NUMBER>          required
├── info_image
│   ├── --image <FILE>                     required
│   └── --output <FILE>                    default: stdout
├── verify_image
│   ├── --image <FILE>                     required
│   ├── --key <KEY>
│   ├── --pass-file <FILE>
│   ├── --expected_chain_partition <PART_NAME:ROLLBACK_SLOT:KEY_PATH>
│   │                                      repeatable
│   ├── --follow_chain_partitions
│   └── --accept_zeroed_hashtree
├── print_partition_digests
│   ├── --image <FILE>                     required
│   ├── --output <FILE>                    default: stdout
│   └── --json
├── calculate_vbmeta_digest
│   ├── --image <FILE>                     required
│   ├── --hash_algorithm <NAME>            default: sha256
│   └── --output <FILE>                    default: stdout
├── calculate_kernel_cmdline
│   ├── --image <FILE>                     required
│   ├── --hashtree_disabled
│   └── --output <FILE>                    default: stdout
├── set_ab_metadata
│   ├── --misc_image <FILE>                required; created if absent
│   └── --slot_data <A:B>                  default: 15:7:0:14:7:0
├── make_atx_certificate
│   ├── --output <FILE>                    default: stdout
│   ├── --subject <FILE>                   required
│   ├── --subject_key <FILE>               required; PEM or DER public key
│   ├── --subject_key_version <NUMBER>     default: current time
│   ├── --subject_is_intermediate_authority
│   ├── --usage <STRING>
│   ├── --authority_key <FILE>
│   ├── --signing_helper <PROGRAM>
│   └── --signing_helper_with_files <PROGRAM>
├── make_atx_permanent_attributes
│   ├── --output <FILE>                    default: stdout
│   ├── --root_authority_key <FILE>        required; PEM or DER public key
│   └── --product_id <FILE>                required; exactly 16 bytes
├── make_atx_metadata
│   ├── --output <FILE>                    default: stdout
│   ├── --intermediate_key_certificate <FILE>
│   │                                      required
│   └── --product_key_certificate <FILE>   required
└── make_atx_unlock_credential
    ├── --output <FILE>                    default: stdout
    ├── --intermediate_key_certificate <FILE>
    │                                      required
    ├── --unlock_key_certificate <FILE>    required
    ├── --challenge <FILE>                 optional; exactly 16 bytes
    ├── --unlock_key <FILE>                required with --challenge
    ├── --signing_helper <PROGRAM>
    └── --signing_helper_with_files <PROGRAM>
```

Commands marked `*` accept these shared options:

```text
Signing and metadata options
├── --algorithm <ALGORITHM>                default: NONE
├── --key <KEY>
├── --signing_helper <PROGRAM>
├── --signing_helper_with_files <PROGRAM>
├── --public_key_metadata <FILE>
├── --rollback_index <NUMBER>              default: 0
├── --rollback_index_location <NUMBER>     default: 0
├── --append_to_release_string <STRING>
├── --prop <KEY:VALUE>                     repeatable
├── --prop_from_file <KEY:PATH>            repeatable
├── --kernel_cmdline <CMDLINE>             repeatable
├── --setup_rootfs_from_kernel <IMAGE>
│   alias: --generate_dm_verity_cmdline_from_hashtree
├── --include_descriptors_from_image <IMAGE>
│                                          repeatable
├── --print_required_libavb_version
├── --chain_partition <PART_NAME:ROLLBACK_SLOT:KEY_PATH>
│                                          repeatable
├── --flags <NUMBER>                       default: 0
├── --set_hashtree_disabled_flag
└── --pass-file <FILE>

Footer options
├── --use_persistent_digest
└── --do_not_use_ab
```

Supported `<ALGORITHM>` values are `NONE`, `SHA256_RSA2048`, `SHA256_RSA4096`, `SHA256_RSA8192`, `SHA512_RSA2048`, `SHA512_RSA4096`, and `SHA512_RSA8192`.

## License

`avbtool.py` and `LICENSE` retain the upstream Apache 2.0 terms.

Copyright 2016, The Android Open Source Project