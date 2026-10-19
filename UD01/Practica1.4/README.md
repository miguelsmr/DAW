# TaskFlow API

Proyecto inicial con FastAPI.

## Cómo arrancarlo

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload
```

La documentación está en http://127.0.0.1:8000/docs

## Endpoints

- `GET /health` → `{"status": "ok"}`
- `GET /version` → `{"version": "1.0.0"}`
- `GET /ping` → `{"message": "pong"}`
