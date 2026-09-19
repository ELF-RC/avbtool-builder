# AVBTOOL Mod

[English](README.md) | 中文

原项目：https://github.com/AndroidBootloader/platform_external_avb

AVBTOOL Mod 是从原版 Android Verified Boot 项目中独立出来的 `avbtool`。它保留上游命令结构，并在进程内完成 RSA 运算、签名校验和内置 FEC 生成。

## 新增改动

AVBTOOL Mod 所做的改动：

- `--pass-file <FILE>`：为签名命令、`extract_public_key` 和 `verify_image` 在进程内读取加密 RSA 密钥的口令。密钥不会被写入解密后的临时文件。
- `--dynamic_partition_size`：为 `add_hash_footer` 和 `add_hashtree_footer` 自动计算分区大小。它不能与 `--partition_size` 或 `--calc_max_image_size` 同时使用。
- 内置 FEC：`add_hashtree_footer` 默认生成与 AOSP 兼容的 FEC 数据及其 4 KiB 尾部，不需要外部 `fec` 工具。
- 进程内加密实现：使用 `cryptography` 解析并签署 PEM 和 DER 格式的 RSA 密钥。`verify_image` 不调用 OpenSSL 即可校验签名。
- Python 3 输出修复：默认写入二进制数据的命令可以直接使用标准输出，不会再出现二进制流 `AttributeError`。

## 环境要求

- Python 3.11 或更新版本
- `cryptography` 包

```sh
python3 -m pip install cryptography
python3 avbtool.py --help
```

可以手动运行 PyInstaller 或 Nuitka 工作流，为 Ubuntu 22.04 amd64 和 arm64 构建独立可执行文件。两个工作流都会打包 `cryptography`。

## 命令树

以下命令树根据当前 `avbtool.py` 中的 `argparse` 命令定义整理。带星号的命令同时接受下方共享参数组中的参数。

```text
avbtool
├── generate_test_image
│   ├── --image_size <NUMBER>              必填
│   ├── --start_byte <NUMBER>              默认：0
│   └── --output <FILE>                    默认：标准输出
├── version
├── extract_public_key
│   ├── --key <FILE>                       必填
│   ├── --output <FILE>                    必填
│   └── --pass-file <FILE>
├── make_vbmeta_image *
│   ├── --output <FILE>
│   └── --padding_size <NUMBER>            默认：0
├── add_hash_footer *
│   ├── --image <FILE>
│   ├── --partition_size <NUMBER>          默认：0
│   ├── --dynamic_partition_size
│   ├── --partition_name <NAME>
│   ├── --hash_algorithm <NAME>            默认：sha256
│   ├── --salt <HEX>                       默认：随机
│   ├── --calc_max_image_size
│   ├── --output_vbmeta_image <FILE>
│   ├── --do_not_append_vbmeta_image
│   └── Footer options
├── append_vbmeta_image
│   ├── --image <FILE>                     必填
│   ├── --partition_size <NUMBER>          必填
│   └── --vbmeta_image <FILE>              必填
├── add_hashtree_footer *
│   ├── --image <FILE>
│   ├── --partition_size <NUMBER>          默认：0
│   ├── --dynamic_partition_size
│   ├── --partition_name <NAME>            默认：空
│   ├── --hash_algorithm <NAME>            默认：sha1
│   ├── --salt <HEX>                       默认：随机
│   ├── --block_size <NUMBER>              默认：4096
│   ├── --do_not_generate_fec
│   ├── --fec_num_roots <NUMBER>           默认：2
│   ├── --calc_max_image_size
│   ├── --output_vbmeta_image <FILE>
│   ├── --do_not_append_vbmeta_image
│   ├── --setup_as_rootfs_from_kernel
│   ├── --no_hashtree
│   └── Footer options
├── erase_footer
│   ├── --image <FILE>                     必填
│   └── --keep_hashtree
├── zero_hashtree
│   └── --image <FILE>                     必填
├── extract_vbmeta_image
│   ├── --image <FILE>                     必填
│   ├── --output <FILE>                    默认：标准输出
│   └── --padding_size <NUMBER>            默认：0
├── resize_image
│   ├── --image <FILE>                     必填
│   └── --partition_size <NUMBER>          必填
├── info_image
│   ├── --image <FILE>                     必填
│   └── --output <FILE>                    默认：标准输出
├── verify_image
│   ├── --image <FILE>                     必填
│   ├── --key <KEY>
│   ├── --pass-file <FILE>
│   ├── --expected_chain_partition <PART_NAME:ROLLBACK_SLOT:KEY_PATH>
│   │                                      可重复
│   ├── --follow_chain_partitions
│   └── --accept_zeroed_hashtree
├── print_partition_digests
│   ├── --image <FILE>                     必填
│   ├── --output <FILE>                    默认：标准输出
│   └── --json
├── calculate_vbmeta_digest
│   ├── --image <FILE>                     必填
│   ├── --hash_algorithm <NAME>            默认：sha256
│   └── --output <FILE>                    默认：标准输出
├── calculate_kernel_cmdline
│   ├── --image <FILE>                     必填
│   ├── --hashtree_disabled
│   └── --output <FILE>                    默认：标准输出
├── set_ab_metadata
│   ├── --misc_image <FILE>                必填；不存在时自动创建
│   └── --slot_data <A:B>                  默认：15:7:0:14:7:0
├── make_atx_certificate
│   ├── --output <FILE>                    默认：标准输出
│   ├── --subject <FILE>                   必填
│   ├── --subject_key <FILE>               必填；PEM 或 DER 公钥
│   ├── --subject_key_version <NUMBER>     默认：当前时间
│   ├── --subject_is_intermediate_authority
│   ├── --usage <STRING>
│   ├── --authority_key <FILE>
│   ├── --signing_helper <PROGRAM>
│   └── --signing_helper_with_files <PROGRAM>
├── make_atx_permanent_attributes
│   ├── --output <FILE>                    默认：标准输出
│   ├── --root_authority_key <FILE>        必填；PEM 或 DER 公钥
│   └── --product_id <FILE>                必填；必须为 16 字节
├── make_atx_metadata
│   ├── --output <FILE>                    默认：标准输出
│   ├── --intermediate_key_certificate <FILE>
│   │                                      必填
│   └── --product_key_certificate <FILE>   必填
└── make_atx_unlock_credential
    ├── --output <FILE>                    默认：标准输出
    ├── --intermediate_key_certificate <FILE>
    │                                      必填
    ├── --unlock_key_certificate <FILE>    必填
    ├── --challenge <FILE>                 可选；必须为 16 字节
    ├── --unlock_key <FILE>                使用 --challenge 时必填
    ├── --signing_helper <PROGRAM>
    └── --signing_helper_with_files <PROGRAM>
```

带 `*` 的命令接受以下共享参数：

```text
签名与元数据参数
├── --algorithm <ALGORITHM>                默认：NONE
├── --key <KEY>
├── --signing_helper <PROGRAM>
├── --signing_helper_with_files <PROGRAM>
├── --public_key_metadata <FILE>
├── --rollback_index <NUMBER>              默认：0
├── --rollback_index_location <NUMBER>     默认：0
├── --append_to_release_string <STRING>
├── --prop <KEY:VALUE>                     可重复
├── --prop_from_file <KEY:PATH>            可重复
├── --kernel_cmdline <CMDLINE>             可重复
├── --setup_rootfs_from_kernel <IMAGE>
│   别名：--generate_dm_verity_cmdline_from_hashtree
├── --include_descriptors_from_image <IMAGE>
│                                          可重复
├── --print_required_libavb_version
├── --chain_partition <PART_NAME:ROLLBACK_SLOT:KEY_PATH>
│                                          可重复
├── --flags <NUMBER>                       默认：0
├── --set_hashtree_disabled_flag
└── --pass-file <FILE>

Footer 参数
├── --use_persistent_digest
└── --do_not_use_ab
```

支持的 `<ALGORITHM>` 为 `NONE`、`SHA256_RSA2048`、`SHA256_RSA4096`、`SHA256_RSA8192`、`SHA512_RSA2048`、`SHA512_RSA4096` 和 `SHA512_RSA8192`。

## 许可证

`avbtool.py` 和 `LICENSE` 保留上游 Apache 2.0 许可条款。

Copyright 2016, The Android Open Source Project