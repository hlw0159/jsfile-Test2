# Final JS Hacking

```
import re
import asyncio
import aiohttp
from aiohttp import ClientSession, ClientTimeout
from urllib.parse import urljoin
from bs4 import BeautifulSoup
import sys
import os
from tqdm.asyncio import tqdm_asyncio

HEADERS = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/119.0.0.0 Safari/537.36 HackeroneBugbountytest"
}

PATTERNS = {
    "Google API Key": r"AIza[0-9A-Za-z-_]{35}",
    "Slack Webhook": r"https://hooks.slack.com/services/T[a-zA-Z0-9]{10}/B[a-zA-Z0-9]{10}/[a-zA-Z0-9]{24}",
    "Firebase URL": r"https://[a-z0-9\-]+\.firebaseio\.com",
    "Heroku API Key": r"heroku[a-zA-Z0-9]{32}",
    "GitHub Token": r"gh[pousr]_[A-Za-z0-9]{36}",
    "Stripe Key": r"sk_live_[0-9a-zA-Z]{24}",
    "JWT Token": r"eyJ[A-Za-z0-9\-_]+\.[A-Za-z0-9\-_]+\.[A-Za-z0-9\-_]+",
    "UUID": r"\b[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}\b",
    "Twilio SID AC": r"AC[a-zA-Z0-9]{32}",
    "Twilio SID SK": r"SK[a-zA-Z0-9]{32}",
    "Twilio SID AP": r"AP[a-zA-Z0-9]{32}",
    "Mapbox Token": r"pk\.[a-z0-9]{60,}",
    "SendGrid API Key": r"SG\.[A-Za-z0-9_-]{22,}\.[A-Za-z0-9_-]{22,}",
    "Mailgun API Key": r"key-[a-z0-9]{32}",
    "OpenWeatherMap API Key": r"[?&]appid=([a-fA-F0-9]{32})",
    "Cloudinary API Key": r"cloudinary://[0-9a-zA-Z]+:[0-9a-zA-Z]+@[0-9a-zA-Z]+",
    "Imgur Client ID": r"Client-ID\s[0-9a-zA-Z]{15,30}",
    "OneSignal App ID": r"[a-f0-9]{8}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{12}",
    "OneSignal REST API Key": r"AAAA[A-Za-z0-9_\-]{7,}"
}

FALSE_POSITIVE_PATTERNS = {
    "UUID": lambda v: v.lower() == "d27cdb6e-ae6d-11cf-96b8-444553540000",
    "OneSignal REST API Key": lambda v: v in ["AAAAAElFTkSuQmCC", "AAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8"]
}

def is_false_positive(key_name, value):
    if len(value) < 15:
        return True
    if key_name in FALSE_POSITIVE_PATTERNS and FALSE_POSITIVE_PATTERNS[key_name](value):
        return True
    return False

VALIDATORS = {}

async def fetch_text(session: ClientSession, url: str) -> str:
    try:
        async with session.get(url, timeout=10) as resp:
            if resp.status == 200:
                return await resp.text()
    except:
        pass
    return ""

async def extract_js_links(session: ClientSession, url: str) -> list:
    html = await fetch_text(session, url)
    if not html:
        return []
    soup = BeautifulSoup(html, "html.parser")
    scripts = soup.find_all("script", src=True)
    return [urljoin(url, s["src"]) for s in scripts if s["src"].endswith(".js")]

async def check_key_validity(session: ClientSession, key_name: str, key_value: str) -> bool:
    validator = VALIDATORS.get(key_name)
    if validator:
        return await validator(session, key_value)
    return True

async def scan_js_for_keys(session: ClientSession, js_url: str) -> list:
    js_content = await fetch_text(session, js_url)
    if not js_content:
        return []
    findings = []
    for key_name, pattern in PATTERNS.items():
        for match in set(re.findall(pattern, js_content)):
            if is_false_positive(key_name, match):
                continue
            valid = await check_key_validity(session, key_name, match)
            findings.append((key_name, match, valid))
    return findings

async def process_url(session: ClientSession, url: str) -> list:
    results = []
    if url.endswith(".js"):
        results.append((url, await scan_js_for_keys(session, url)))
    else:
        for js_url in await extract_js_links(session, url):
            results.append((js_url, await scan_js_for_keys(session, js_url)))
    return results

async def main(input_arg):
    urls = [input_arg] if not os.path.isfile(input_arg) else [line.strip() for line in open(input_arg) if line.strip()]
    timeout = ClientTimeout(total=15)
    async with ClientSession(headers=HEADERS, timeout=timeout) as session:
        results = []
        for url in tqdm_asyncio(urls, desc="Scanning URLs"):
            results.extend(await process_url(session, url))
    for js_url, findings in results:
        print(f"\n[+] 검사 대상: {js_url}")
        if findings:
            for key_name, key, valid in findings:
                print(f"  [!] {key_name} 발견: {key} ({'✔ 유효' if valid else '✘ 무효 또는 제한됨'})")
        else:
            print("  [-] 민감 키 없음")

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print(f"사용법: python {sys.argv[0]} <URL 또는 파일 경로>")
        sys.exit(1)
    asyncio.run(main(sys.argv[1]))
```
