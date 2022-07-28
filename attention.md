# 注意事項

### 此為laravel9使用環境

### 若為了laravel升版使用，請注意以下幾點
- 須先刪除 `composer.lock`
- 刪除 `vendor` 資料夾
- 執行 `composer install --ignore-platform-req=ext-bcmath`
- 建議也重新 `npm install` 