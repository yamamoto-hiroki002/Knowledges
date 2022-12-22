# OpenSSHを使ったSSHの設定
* 参考：https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse
* 注意：下記の手順ではWindowsの機能追加でOpenSSHを入れているが、WindowsServer2019より前のWindowsServerだとGithubからソースを落としてきて、設定する必要あり（https://www.server-world.info/query?os=Windows_Server_2016&p=openssh）

## OpenSSHが利用可能かの確認
```
>Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH*'

インストールされていない場合、下記の値をリターンする
Name  : OpenSSH.Client~~~~0.0.1.0
State : NotPresent

Name  : OpenSSH.Server~~~~0.0.1.0
State : NotPresent
```

入っていない場合、下記のコマンドを使用して、機能をインストールする

```
# Install the OpenSSH Client
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0

# Install the OpenSSH Server
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

実行すると下記のような出力を返す
Path          :
Online        : True
RestartNeeded : False
```

## SSH有効化 ##
```
### OpenSSHの機能追加（WindowsServer2016以前など機能追加で追加できない場合は手動で入れる）
Add-WindowsCapability -Online -Name 'OpenSSH.Server~~~~0.0.1.0'
Add-WindowsCapability -Online -Name 'OpenSSH.Client~~~~0.0.1.0'

### 以降の手順は機能追加で入れた場合・入れなかった場合両方で共通の処理
### SSHで使用するTCP 22ポートの解放のルールを追加
New-NetFirewallRule -Protocol TCP -LocalPort 22 -Direction Inbound -Action Allow -DisplayName SSH
### OpenSSHサーバーサービスの起動
Start-Service -Name "sshd"
### ↑サービスを自動起動に変更
Set-Service -Name "sshd" -StartupType Automatic
### SSH接続時のシェルをPowershellに設定
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -PropertyType String -Force
```
#### Add-WindowsCapability
Windowsに機能を追加するコマンド（https://docs.microsoft.com/en-us/powershell/module/dism/add-windowscapability?view=windowsserver2022-ps）

#### New-NetFirewallRule
ファイアウォールルール規則の追加（https://docs.microsoft.com/en-us/powershell/module/netsecurity/new-netfirewallrule?view=windowsserver2022-ps）

#### Start-Service -Name "sshd"
OpenSSHサーバーの起動

#### New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -PropertyType String -Force
SSHで接続した際のシェルをPowershellに設定
(https://docs.microsoft.com/ja-jp/windows-server/administration/openssh/openssh_server_configuration)


## 公開鍵設置（管理者用）
3行目の"testKey"の部分はクライアントPCで作成した公開鍵の中身を記載（直接ファイルに書き込むと文字コードとかずれるのでコマンドのほうがベター）
```
$authKeyPath = "$env:ProgramData\ssh\administrators_authorized_keys"
New-Item $authKeyPath
Write-Output "testKey" | Out-File -FilePath $authKeyPath -Encoding ascii
```

### 管理者用のauthrorized_keysの登録について（Windowsの特殊な設定）
Windowsの特殊な設定として、標準ユーザと管理者ユーザ（ローカル管理者グループメンバー）であるかによって、公開鍵の登録場所（登録ファイルが異なる）
* authroized_keys：標準ユーザ用
* administrators_authorised_keys：管理者ユーザ用  
  参考：https://docs.microsoft.com/ja-jp/windows-server/administration/openssh/openssh_keymanagement#deploying-the-public-key


## アクセス権変更
```
$acl = Get-Acl $authKeyPath
$acl.SetAccessRuleProtection($true,$true)
$removeRule = $acl.Access | Where-Object { $_.IdentityReference -eq 'NT AUTHORITY\Authenticated Users' }
$acl.RemoveAccessRule($removeRule)
$acl | Set-Acl -Path $authKeyPath
```
#### SetAccessRuleProtection($true,$true)
上位フォルダからアクセス規則を継承、継承されたアクセス規則を保持する場合にTrueに設定
https://docs.microsoft.com/ja-jp/dotnet/api/system.security.accesscontrol.objectsecurity.setaccessruleprotection?view=net-6.0


## ユーザ毎のauthorized_keysに変更
```
Copy-Item $env:ProgramData\ssh\sshd_config $env:ProgramData\ssh\sshd_config.org
$(Get-Content "$env:ProgramData\ssh\sshd_config") -replace "Match","#Match" |Out-File -Encoding Default $env:ProgramData\ssh\sshd_config
$(Get-Content "$env:ProgramData\ssh\sshd_config") -replace "AuthorizedKeysFile","#AuthorizedKeysFile" |Out-File -Encoding Default $env:ProgramData\ssh\sshd_config

$(Get-Content "$env:ProgramData\ssh\sshd_config") -replace "#PubkeyAuthentication","PubkeyAuthentication" |Out-File -Encoding Default $env:ProgramData\ssh\sshd_config
```

## プロンプト変更  オプション設定
```
New-Item $HOME\Documents\WindowsPowerShell -ItemType Directory
New-Item $HOME\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1
Set-Item function:Global:prompt {"PS:[" + $(hostname) +"] "+ (Split-Path (Get-Location) -Leaf) +"/ >"} -force
Write-Output 'function prompt() {"PS:[" + $(hostname) +"] "+ (Split-Path (Get-Location) -Leaf) +"/ >"}' >> $HOME\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1
Set-ItemProperty -Path $HOME\Documents\WindowsPowerShell -Name Attributes -Value Hidden## SSH有効化 ##
#Get-WindowsCapability -Online | ? Name -like 'OpenSSH*'
Add-WindowsCapability -Online -Name 'OpenSSH.Server~~~~0.0.1.0'
Add-WindowsCapability -Online -Name 'OpenSSH.Client~~~~0.0.1.0'
New-NetFirewallRule -Protocol TCP -LocalPort 22 -Direction Inbound -Action Allow -DisplayName SSH
Start-Service -Name "sshd"
Set-Service -Name "sshd" -StartupType Automatic
# ssh Administrator@localhost -t powershell
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -PropertyType String -Force
```

## 公開鍵設置（標準ユーザ用）
```
$authKeyPath = "$env:ProgramData\ssh\administrators_authorized_keys"
New-Item $authKeyPath
Write-Output "testKey" | Out-File -FilePath $authKeyPath -Encoding ascii
```


## アクセス権変更
```
$acl = Get-Acl $authKeyPath
$acl.SetAccessRuleProtection($true,$true)
$removeRule = $acl.Access | Where-Object { $_.IdentityReference -eq 'NT AUTHORITY\Authenticated Users' }
$acl.RemoveAccessRule($removeRule)
$acl | Set-Acl -Path $authKeyPath
```

## ユーザ毎のauthorized_keysに変更 
```
Copy-Item $env:ProgramData\ssh\sshd_config $env:ProgramData\ssh\sshd_config.org
$(Get-Content "$env:ProgramData\ssh\sshd_config") -replace "Match","#Match" |Out-File -Encoding Default $env:ProgramData\ssh\sshd_config
$(Get-Content "$env:ProgramData\ssh\sshd_config") -replace "AuthorizedKeysFile","#AuthorizedKeysFile" |Out-File -Encoding Default $env:ProgramData\ssh\sshd_config
```


## プロンプト変更  $PROFILE (バージョン確認)
```
New-Item $HOME\Documents\WindowsPowerShell -ItemType Directory
New-Item $HOME\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1
Set-Item function:Global:prompt {"PS:[" + $(hostname) +"] "+ (Split-Path (Get-Location) -Leaf) +"/ >"} -force
Write-Output 'function prompt() {"PS:[" + $(hostname) +"] "+ (Split-Path (Get-Location) -Leaf) +"/ >"}' >> $HOME\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1
Set-ItemProperty -Path $HOME\Documents\WindowsPowerShell -Name Attributes -Value Hidden
```

## クライアントPCのSSH_configに設定追加
```
Host JPPCCS000067
     HostName 132.182.239.126
     User pcc-ad\JPPCCZSQL
     Port 22
     IdentityFile C:\Users\4076236\.ssh\id_kdev
```

## ssh接続確認
Powershellを開いて、下記コマンドを実行
```
ssh ssh_configに記載したホスト名
ex) ssh JPPCCS000067
```
パスワードが求められる場合は、公開鍵認証の設定に失敗している可能性大。下記を確認
* サーバ側のsshd_configで「PubkeyAuthentication Yes」となっているか確認。＃などでコメントアウトされていないか
* 公開鍵を設置しているフォルダのアクセス権設定がおかしい



