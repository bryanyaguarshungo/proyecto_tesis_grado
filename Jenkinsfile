# ❺ Script de humo
'#!/usr/bin/env bash'      | Out-File -Encoding ascii app\tests\smoke.sh
'curl -s -o NUL -w %{{http_code}} http://198.154.99.201:30080' >> app\tests\smoke.sh
git add app\tests\smoke.sh
git add app\Dockerfile

# ❻ Kustomization
@"
resources:
  - base/
images:
  - name: moodle
    newName: docker.io/bryanyaguarshungo/moodle
    newTag: latest
"@ | Out-File -Encoding ascii infra\kustomization.yaml
git add infra\kustomization.yaml
