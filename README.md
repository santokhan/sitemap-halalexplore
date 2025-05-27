# Halalexplore `sitemap.xml`

```python
from fastapi import FastAPI, HTTPException, Depends
from fastapi.responses import JSONResponse
from sqlalchemy.orm import Session
import httpx

from .database import SessionLocal
from .models import SitemapEntry

app = FastAPI()

REMOTE_API_URL = "https://example.com/api/hotels"

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Admin-only endpoint
# This endpoint is intended to be used by the admin to fetch hotel data from a remote API
# and regenerate the sitemap stored in the database.
@app.post("/sitemap/generate", tags=["Admin"])
async def generate_sitemap(db: Session = Depends(get_db)):
    try:
        async with httpx.AsyncClient() as client:
            res = await client.get(REMOTE_API_URL)
            res.raise_for_status()
            hotels = res.json()
    except Exception as e:
        raise HTTPException(500, f"Fetch failed: {e}")

    db.query(SitemapEntry).delete()
    db.bulk_save_objects([
        SitemapEntry(loc=f"https://yourdomain.com/hotels/{h['slug']}")
        for h in hotels
    ])
    db.commit()
    return {"status": "generated", "count": len(hotels)}

# Frontend/public endpoint
# This endpoint is intended to be used by the frontend or crawlers to serve the generated sitemap in JSON format.
@app.get("/sitemap.json", tags=["Public"])
def serve_sitemap(db: Session = Depends(get_db)):
    entries = db.query(SitemapEntry).all()
    if not entries:
        raise HTTPException(503, "Sitemap not generated")
    return JSONResponse({
        "sitemap": [
            {"loc": e.loc, "changefreq": e.changefreq, "priority": e.priority}
            for e in entries
        ]
    })
```
