#!/usr/bin/env bash
set -euo pipefail

# ---------- 1. نصب پیش‌نیازها ----------
echo "📦 نصب Docker و mc ..."
if ! command -v docker >/dev/null 2>&1; then
  sudo apt-get update
  sudo apt-get install -y ca-certificates curl gnupg lsb-release wget
  sudo install -m 0755 -d /etc/apt/keyrings
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
  echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  sudo apt-get update
  sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
  sudo usermod -aG docker $USER
fi
wget -q https://dl.min.io/client/mc/release/linux-amd64/mc && chmod +x mc && sudo mv mc /usr/local/bin/

# ---------- 2. ساخت شبکه ----------
docker network create citynet >/dev/null 2>&1 || true

# ---------- 3. فایل env ----------
cat > .env <<'EOF'
POSTGRES_PASSWORD=Pass_pg!
KEYCLOAK_USER=admin
KEYCLOAK_PASSWORD=Pass_kc!
MINIO_ROOT_USER=admin
MINIO_ROOT_PASSWORD=Pass_s3!
LITELLM_API_KEY=admin-key
OLLAMA_MODEL=llama3
EOF

# ---------- 4. docker-compose ----------
cat > docker-compose.yml <<'YML'
version: "3.8"
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    networks: [citynet]
  keycloak:
    image: quay.io/keycloak/keycloak:24.0
    command: ["start-dev"]
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/postgres
      KC_DB_USERNAME: postgres
      KC_DB_PASSWORD: ${POSTGRES_PASSWORD}
      KEYCLOAK_ADMIN: ${KEYCLOAK_USER}
      KEYCLOAK_ADMIN_PASSWORD: ${KEYCLOAK_PASSWORD}
    depends_on: [postgres]
    networks: [citynet]
  nats:
    image: nats:2.10
    command: ["-js", "-sd", "/data"]
    networks: [citynet]
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
    networks: [citynet]
  ollama:
    image: ollama/ollama:latest
    environment:
      OLLAMA_KEEP_ALIVE: 24h
    networks: [citynet]
    entrypoint: ["/bin/sh","-c","ollama serve & sleep 3 && ollama pull ${OLLAMA_MODEL} && tail -f /dev/null"]
  litellm:
    image: ghcr.io/berriai/litellm:main
    environment:
      LITELLM_API_KEY: ${LITELLM_API_KEY}
      LITELLM_MODELS: "ollama/${OLLAMA_MODEL}"
      OLLAMA_BASE_URL: http://ollama:11434
    depends_on: [ollama]
    networks: [citynet]
networks:
  citynet:
    external: true
YML

# ---------- 5. بالا آوردن سرویس‌ها ----------
echo "🚀 راه‌اندازی سرویس‌ها..."
docker compose up -d
sleep 10

# ---------- 6. فایل ورودی نمونه ----------
echo "🌐 ساخت فایل ورودی..."
echo "این یک متن آزمایشی برای خلاصه‌سازی است." > real_input.txt

# ---------- 7. ارسال به مدل و ذخیره خروجی ----------
echo "🤖 پردازش با LLM..."
SUMMARY=$(curl -s -X POST http://localhost:4000/v1/chat/completions \
-H "Content-Type: application/json" \
-H "Authorization: Bearer admin-key" \
-d "{
  \"model\": \"ollama/llama3\",
  \"messages\": [
    {\"role\":\"system\",\"content\":\"متن را کوتاه و شفاف خلاصه کن.\"},
    {\"role\":\"user\",\"content\":\"$(cat real_input.txt)\"}
  ]
}" | jq -r '.choices[0].message.content')

echo "$SUMMARY" > summary_real.txt

# ---------- 8. بارگذاری در MinIO ----------
echo "💾 بارگذاری خروجی در MinIO..."
mc alias set localminio http://localhost:9001 admin Pass_s3!
mc mb localminio/reports || true
mc cp summary_real.txt localminio/reports/

echo "✅ سناریو کامل شد. خلاصه در Bucket «reports» موجود است."
