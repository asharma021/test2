Select-String -Path "src/extract_load/common/cli.py" `
  -Pattern "tables.yml|config_path|mapping_path|--config" `
  -Context 3,5
