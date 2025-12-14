# PXEブート切替Playbookの説明

## 目的
Redfish対応のBMCにREST API (Redfish) でPATCH/POSTを送り、サーバのBIOSブート設定をHDDからPXEブートへ切り替えるAnsible Playbookです。`boot_enable`を`Continuous`に設定すると常時PXE優先、`Once`にすると次回1回のみPXE起動になります。

## 前提条件
- Ansible実行ホストからBMC (Redfish) にHTTPSで到達できること
- 対象機がRedfish 1.0以降をサポートしていること
- `ansible.builtin.uri`でBMCへHTTPSアクセスできること (自己署名証明書の場合は`validate_certs: false`を設定)

## 変数一覧
- `bmc_host`: BMCのホスト名またはIP
- `bmc_port`: BMCのHTTPS/HTTPポート (例: dmtf/redfish-interface-emulator はデフォルト5000)
- `bmc_scheme`: `https` または `http` (emulatorは`http`)
- `bmc_user`: BMCユーザ名
- `bmc_password`: BMCパスワード (ログ出力は`no_log: true`でマスク)
- `system_id`: Redfish上のSystem ID (例: `1`や`System.Embedded.1`)
- `boot_enable`: `Continuous`(恒久) または `Once`(次回のみ)
- `boot_mode`: `Uefi` または `Legacy`
- `reboot_after`: 変更後にGraceful再起動する場合は`true`
- `validate_certs`: BMC証明書を検証する場合は`true` (自己署名なら`false`)

## tasks内のポイント
- `Set PXE boot via Redfish REST (PATCH Boot)`: `PATCH /redfish/v1/Systems/<system_id>` にBootオーバーライドを設定し、起動デバイスをPXE (`BootSourceOverrideTarget: Pxe`) に変更します。`BootSourceOverrideEnabled`で恒久/1回限りを指定し、`BootSourceOverrideMode`でUEFI/Legacyを指定します。
- `Reboot system to apply PXE boot change (optional)`: `POST /redfish/v1/Systems/<system_id>/Actions/ComputerSystem.Reset` に `ResetType: GracefulRestart` を送信し、設定反映のために再起動します。`reboot_after`が`true`のときだけ実行されます。

## 実行例
```bash
ansible-playbook playbooks/pxe_boot.yml \
  -e "bmc_scheme=http bmc_port=5000 bmc_host=127.0.0.1 bmc_user=admin bmc_password='pass' system_id=1 boot_enable=Continuous reboot_after=false validate_certs=false"
```

## 動作の流れ
1. `PATCH <scheme>://<BMC>:<port>/redfish/v1/Systems/<system_id>` に対しBootオーバーライドをPXE (`Pxe`) に設定
2. `reboot_after`が`true`の場合、`POST <scheme>://<BMC>:<port>/redfish/v1/Systems/<system_id>/Actions/ComputerSystem.Reset` に `ResetType: GracefulRestart` を送信し再起動

## 参考URL
- Ansible `community.general.redfish_command` ドキュメント: https://docs.ansible.com/ansible/latest/collections/community/general/redfish_command_module.html
- DMTF Redfish Schema (Boot要素の定義): https://redfish.dmtf.org/schemas/v1/ComputerSystem.v1_20_0.json
