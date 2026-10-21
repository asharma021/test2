  script:
    - |
      $paths = @(
        @{ Name = "Share root"; Path = '\\thqpcelerra01\ttb_common' },
        @{ Name = "Data sharing"; Path = '\\thqpcelerra01\ttb_common\OA_Data_Sharing' },
        @{ Name = "OA directory"; Path = '\\thqpcelerra01\ttb_common\OA_Data_Sharing\OA' },
        @{ Name = "DEV environment"; Path = '\\thqpcelerra01\ttb_common\OA_Data_Sharing\OA\dbt_dev_env' },
        @{ Name = "EL environment file"; Path = '\\thqpcelerra01\ttb_common\OA_Data_Sharing\OA\dbt_dev_env\extract_load.env' }
      )

      foreach ($item in $paths) {
        if (Test-Path -LiteralPath $item.Path) {
          Write-Output "$($item.Name): ACCESSIBLE"
        } else {
          Write-Output "$($item.Name): NOT ACCESSIBLE"
        }
      }
