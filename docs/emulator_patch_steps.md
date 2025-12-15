# Redfish Emulator で Boot を PATCH 可能にする手順まとめ

ローカルにリポジトリを取得・修正し、PXEブート変更のテストができるエミュレータを立てるための手順です。

## 1. リポジトリをクローン
```bash
git clone https://github.com/DMTF/Redfish-Interface-Emulator.git
cd Redfish-Interface-Emulator
```

## 2. `computer_system.py` を修正して Boot を PATCH 許可
対象: `api_emulator/redfish/ComputerSystem_api.py`

- 変更前（例）
```python
    # patch for systems
    # def patch(self, ident):
    #     logging.info('ComputerSystemAPI PATCH called for {ident}')
    #     patch_data = request.get_json(force=True)

    #     try:
            
    #         if ident not in members:
    #             logging.error(f"System {ident} not found in members")
    #             return {"error": f"{ident} not found."}, 404
            
    #         updatable_fields = ["Name", "HostName", "AssetTag", "SerialNumber", "UUID"]
    #         updated_fields = {}

    #         for key in patch_data:
    #             for key in patch_data:
    #                 if key not in updatable_fields:
    #                     return {
    #                         "error": f"Field '{key}' is not updatable"
    #                     }, 400
    #             if key in updatable_fields and key in members[ident]:
    #                 if key == "UUID":
    #                     try:
    #                         new_uuid=uuid.UUID(patch_data[key])
    #                     except ValueError:
    #                         return{
    #                             "error": "Invalid UUID format"
    #                         }, 400
                        
    #                     for sys_id, sys_data in members.items():
    #                         if sys_id != ident and sys_data.get("UUID") == str(new_uuid):
    #                             return {"error": f"UUID {new_uuid} is already used by {sys_id}."}, 400
                        
    #                 members[ident][key] = patch_data[key]
    #                 updated_fields[key] = patch_data[key]
            
    #         return members[ident], 200
        
    #     except Exception as e:
    #         logging.error(f"Error during patch for {ident}: {str(e)}")
    #         traceback.print_exc()
    #         return jsonify({"error": f"An error occurred: {str(e)}"}), 500
```

- 変更後（例）
```python
   def patch(self, ident):
        logging.info(f'ComputerSystemAPI PATCH called for {ident}')
        patch_data = request.get_json(force=True)

        try:
            if ident not in members:
                logging.error(f"System {ident} not found in members")
                return {"error": f"{ident} not found."}, 404

            # PATCHで許可するフィールド一覧
            updatable_fields = [
                "Name", "HostName", "AssetTag", "SerialNumber", "UUID", "Boot"
            ]

            # Boot内の許容キーと値
            allowable_boot_keys = {
                "BootSourceOverrideEnabled": ["Disabled", "Once", "Continuous"],
                "BootSourceOverrideMode": ["UEFI", "Legacy", "Uefi", "LegacyBios"],
                "BootSourceOverrideTarget": ["Pxe", "Hdd", "None", "Usb", "Cd", "Floppy", "UefiHttp"]
            }

            updated_fields = {}

            for key, value in patch_data.items():
                if key not in updatable_fields:
                    return {
                        "error": f"Field '{key}' is not updatable"
                    }, 400

                if key == "UUID":
                    try:
                        new_uuid = uuid.UUID(value)
                    except ValueError:
                        return {
                            "error": "Invalid UUID format"
                        }, 400

                    for sys_id, sys_data in members.items():
                        if sys_id != ident and sys_data.get("UUID") == str(new_uuid):
                            return {"error": f"UUID {new_uuid} is already used by {sys_id}."}, 400

                elif key == "Boot":
                    if "Boot" not in members[ident]:
                        members[ident]["Boot"] = {}
                    for boot_key, boot_val in value.items():
                        if boot_key not in allowable_boot_keys:
                            return {"error": f"Invalid Boot key: {boot_key}"}, 400
                        if boot_val not in allowable_boot_keys[boot_key]:
                            return {"error": f"Invalid value '{boot_val}' for {boot_key}"}, 400
                        members[ident]["Boot"][boot_key] = boot_val
                    updated_fields["Boot"] = value

                else:
                    members[ident][key] = value
                    updated_fields[key] = value

            return members[ident], 200

        except Exception as e:
            logging.error(f"Error during patch for {ident}: {str(e)}")
            traceback.print_exc()
            return jsonify({"error": f"An error occurred: {str(e)}"}), 500
```

## 3. ローカルでコンテナをビルド
```bash
docker build -t redfish-emu-local .
```

## 4. ローカルで起動（5000番で待ち受け）
```bash
docker run --rm -it -p 5000:5000 --name redfish-emu-local redfish-emu-local -port 5000
```
- 5000 が使用中なら `-p 5001:5000` に変更し、後述の `bmc_port` も合わせる。

## 5. Playbook を実行
- ホストに Ansible がある場合
```bash
ansible-playbook playbooks/pxe_boot.yml \
  -e "bmc_scheme=http bmc_host=127.0.0.1 bmc_port=5000 bmc_user=admin bmc_password='pass' system_id=System-1 boot_enable=Continuous reboot_after=false validate_certs=false"
```

- Ansible が無い場合（コンテナ内で pip インストール）
```bash
 $extraVars = '{"reboot_after": true, "bmc_scheme": "http", "bmc_host": "host.docker.internal", "bmc_port": 5000, "bmc_user": "admin", "bmc_password": "pass", "system_id": "System-1", "boot_enable": "Continuous", "validate_certs": false}'

docker run --rm -it `  -v "${PWD}:/workspace" `  -w /workspace `  python:3.11-slim `  /bin/sh -c "pip install --no-cache-dir ansible && ansible-playbook playbooks/pxe_boot.yml -e '$extraVars'"
```
- エミュレータを別ポート公開した場合は `bmc_port` を合わせる。


## 確認コマンド
- PowerShellでRedfish APIをGET
```
Invoke-RestMethod -Method GET -Uri "http://localhost:5000/redfish/v1/Systems/System-1"  
```

## 確認ポイント
- エミュレータのログに 4xx が出ないこと。
- Playbook が 200/204 で成功すること（再起動タスクは `reboot_after` が `true` のときのみ実行）。
- `BootSourceOverride*` の許容値が実装に合っていること（許可値にない文字列で 400 になる場合は許容リストを調整）。

## 参考URL
- Redfish Interface Emulator リポジトリ: https://github.com/DMTF/Redfish-Interface-Emulator
- テンプレート例 `computer_system.py`: https://github.com/DMTF/Redfish-Interface-Emulator/blob/main/api_emulator/redfish/templates/computer_system.py
- Redfish ComputerSystem スキーマ: https://redfish.dmtf.org/schemas/v1/ComputerSystem.v1_20_0.json
