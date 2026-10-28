Select-String -Path "src/extract_load/common/connections.py" `
  -Pattern "def source|self.connect_source|profile" `
  -Context 5,10
