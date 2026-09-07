# ใช้ Python ทำการตรวจสอบ Commit ล่าสุดใน Github เพื่อสั่งให้ Gitea ทำการ Sync ข้อมูลล่าสุดมา เพื่อสั่ง Jenkins ทำงาน
Python จะทำการวิ่งไปเช็ค commit hash ระหว่าง GitHub และ Gitea ผ่าน Git CLI หรือ REST API จากนั้นสั่ง Gitea API ให้ทำการ Sync Mirror ทันที และส่ง webhook ไป Trigger Jenkins Pipeline

## หลักการทำงาน
1. ใช้ git ls-remote เช็ค remote hash ของ GitHub และ Gitea โดยตรงโดยไม่ต้อง clone repo ลงเครื่อง
2. ตรวจสอบ 4 branches: prod-storage, prod-common, prod-api, prod-back-offfice
3. หากมี branch ใด branch หนึ่ง hash ไม่ตรงกัน (GitHub มี commit ใหม่):
  - เรียก API ของ Gitea: POST /api/v1/repos/{owner}/{repo}/mirror-sync เพื่อสั่ง Sync ทันที
  - ส่ง HTTP Request ไปยัง Jenkins Remote Build Trigger URL

```python
import subprocess
import time
import requests
import sys
from datetime import datetime

# --- Configuration ---
CHECK_INTERVAL_SECONDS = 300  # ทุก 5 นาที

# Repositories
GITHUB_REMOTE_URL = "https://github.com/[Git Owner]/[Repo Name].git"
GITEA_REMOTE_URL = "https://gitea.example.com/[Git Owner]/[Repo Name].git"

# Target Branches
TARGET_BRANCHES = [
    "prod-api",
    "prod-front-office",
    "prod-back-offfice"
]

# Gitea Config
GITEA_BASE_URL = "[http://10.3.23.139:3000](https://gitea.example.com)"
GITEA_OWNER = "[Git Owner]"
GITEA_REPO = "[Repo Name]"
# สร้างได้ที่: Gitea Settings -> Applications -> Generate New Token
GITEA_TOKEN = "[Your Gitea API Token]"

# Jenkins Config
JENKINS_URL = "[Jenkins URL]"
JENKINS_JOB_NAME = "[Job name build]"
JENKINS_USER = "[Jenkins Username]"
JENKINS_API_TOKEN = "[Jenkins User API Token]"
JENKINS_JOB_TOKEN = "[Configured Job Auth Token]"


def get_remote_hashes(remote_url: str) -> dict[str, str]:
    """ดึง HEAD commit hash ของแต่ละ branch โดยไม่ต้อง clone repo ลงเครื่อง"""
    cmd = ["git", "ls-remote", "--heads", remote_url]
    res = subprocess.run(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True, check=True)
    
    branch_map = {}
    for line in res.stdout.strip().splitlines():
        if not line:
            continue
        commit_hash, ref = line.split()
        branch_name = ref.replace("refs/heads/", "")
        branch_map[branch_name] = commit_hash
    return branch_map


def trigger_gitea_sync() -> bool:
    """สั่ง Gitea Migration Mirror ให้ทำการ Sync ทันทีผ่าน REST API"""
    url = f"{GITEA_BASE_URL}/api/v1/repos/{GITEA_OWNER}/{GITEA_REPO}/mirror-sync"
    headers = {
        "Authorization": f"token {GITEA_TOKEN}",
        "Accept": "application/json"
    }
    
    response = requests.post(url, headers=headers, timeout=30)
    if response.status_code in [200, 202]:
        return True
    print(f"Failed to sync Gitea: {response.status_code} - {response.text}", file=sys.stderr)
    return False


def trigger_jenkins_build(updated_branches: list[str]) -> bool:
    """Trigger Jenkins pipeline พร้อมส่ง parameter branch ที่อัปเดต"""
    # กรณี Parameterized Build:
    url = f"{JENKINS_URL}/job/{JENKINS_JOB_NAME}/buildWithParameters"
    params = {
        "token": JENKINS_JOB_TOKEN,
        "UPDATED_BRANCHES": ",".join(updated_branches)
    }
    
    # กรณี Non-parameterized build ให้เปลี่ยน URL เป็น:
    # url = f"{JENKINS_URL}/job/{JENKINS_JOB_NAME}/build?token={JENKINS_JOB_TOKEN}"
    # params = {}

    response = requests.post(
        url,
        params=params,
        auth=(JENKINS_USER, JENKINS_API_TOKEN),
        timeout=30
    )
    if response.status_code in [200, 201]:
        return True
    print(f"Failed to trigger Jenkins: {response.status_code} - {response.text}", file=sys.stderr)
    return False


def check_and_process():
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    try:
        github_hashes = get_remote_hashes(GITHUB_REMOTE_URL)
        gitea_hashes = get_remote_hashes(GITEA_REMOTE_URL)

        updated_branches = []
        for branch in TARGET_BRANCHES:
            gh_hash = github_hashes.get(branch)
            gt_hash = gitea_hashes.get(branch)

            if gh_hash and gh_hash != gt_hash:
                print(f"[{now}] Branch '{branch}' updated on GitHub ({gt_hash[:7] if gt_hash else 'None'} -> {gh_hash[:7]})")
                updated_branches.append(branch)

        if updated_branches:
            print(f"[{now}] Updates detected on: {updated_branches}")
            
            # 1. สั่ง Gitea sync
            print(f"[{now}] Triggering Gitea mirror-sync...")
            if trigger_gitea_sync():
                print(f"[{now}] Gitea mirror-sync triggered successfully.")
                
                # รอให้ Gitea ดึง repo เสร็จสิ้นชั่วขณะก่อนบอก Jenkins
                time.sleep(15)

                # 2. ยิง webhook ไปยัง Jenkins
                print(f"[{now}] Triggering Jenkins build...")
                if trigger_jenkins_build(updated_branches):
                    print(f"[{now}] Jenkins build triggered successfully.")
        else:
            print(f"[{now}] No changes detected on watched branches.")

    except Exception as e:
        print(f"[{now}] Error: {str(e)}", file=sys.stderr)


def main():
    print("Starting GitHub-Gitea-Jenkins Sync Watcher...")
    while True:
        check_and_process()
        time.sleep(CHECK_INTERVAL_SECONDS)


if __name__ == "__main__":
    main()
```

## การตั้งค่าระบบที่เกี่ยวข้อง
1. Gitea API Token
   - เข้า Gitea ด้วย user ที่มีสิทธิ์ admin หรือสิทธิ์ write ใน repo ที่ต้องการ
   - ไปที่ Settings -> Applications -> Generate New Token ติ๊กสิทธิ์ repo แล้วนำ token มาใส่ใน
2. GitHub Credentials (กรณี repo เป็น Private) `GITEA_TOKEN`
   - หาก repo `https://github.com/[Git Owner]/[Repo Name].git` บน GitHub เป็น private ให้เพิ่ม Personal Access Token (PAT) ไว้ใน URL เช่น:
```python
GITHUB_REMOTE_URL = "https://<Personal Access Token (PAT)>@github.com/[Git Owner]/[Repo Name].git"
```
หรือคอนฟิก SSH key ไว้ในเครื่องที่รันสคริปต์ แล้วใช้ URL รูปแบบ git@github.com:...
3. Jenkins Remote Trigger
- ไปที่ Job ของ Jenkins -> ติ๊ก Trigger builds remotely (e.g., from scripts)
- ตั้งค่า Authentication Token แล้วนำค่านั้นมาใส่ใน JENKINS_JOB_TOKEN
- ไปที่ User Profile ใน Jenkins -> Configure -> API Token -> Add new Token นำมาใส่ใน JENKINS_API_TOKEN

## รันเป็น Systemd Service
ช่วยให้ระบบ auto-restart เองหาก script crash หรือเซิร์ฟเวอร์ reboot
1. สร้างไฟล์ service definition:
```bash
sudo nano /etc/systemd/system/git-sync-watcher.service
```
2. วางเนื้อหาคอนฟิก:
```bash
[Unit]
Description=GitHub to Gitea Sync and Jenkins Trigger Service
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/path/to/script_directory
ExecStart=/path/to/script_directory/venv/bin/python3 /path/to/script_directory/sync_mirror_and_trigger_jenkins.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```
3. สั่งเปิดและเริ่มทำงาน:
```bash
sudo systemctl daemon-reload
sudo systemctl enable git-sync-watcher
sudo systemctl start git-sync-watcher

# ดูสถานะและ log
sudo systemctl status git-sync-watcher
journalctl -u git-sync-watcher -f
```
