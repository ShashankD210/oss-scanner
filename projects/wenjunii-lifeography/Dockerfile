FROM node:24-bookworm-slim
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates git python3 chromium \
    && rm -rf /var/lib/apt/lists/*
COPY . /src
WORKDIR /src
# This static site has no package manager or build step. Cache externally linked
# JavaScript/styles for offline inspection without rewriting the source pages.
RUN python3 - <<'PY'
import hashlib
import json
from html.parser import HTMLParser
from pathlib import Path
from urllib.parse import urlsplit
from urllib.request import Request, urlopen

urls = set()
class Assets(HTMLParser):
    def handle_starttag(self, tag, attrs):
        attrs = dict(attrs)
        url = attrs.get('src') if tag == 'script' else attrs.get('href') if tag == 'link' and attrs.get('rel') == 'stylesheet' else None
        if url and url.startswith(('https://', 'http://', '//')):
            urls.add('https:' + url if url.startswith('//') else url)

for page in Path('/src').rglob('*.html'):
    parser = Assets()
    parser.feed(page.read_text(encoding='utf-8', errors='replace'))
cache = Path('/opt/oss-scanner-web-cache')
cache.mkdir(parents=True)
manifest = {}
for url in sorted(urls):
    fetch_url = 'https://' + url[7:] if url.startswith('http://') else url
    filename = hashlib.sha256(url.encode()).hexdigest() + ('.css' if urlsplit(url).path.endswith('.css') else '.js')
    request = Request(fetch_url, headers={'User-Agent': 'oss-scanner-build/1.0'})
    with urlopen(request, timeout=60) as response:
        content = response.read(32 * 1024 * 1024 + 1)
    if len(content) > 32 * 1024 * 1024:
        raise RuntimeError('External asset exceeds 32 MiB: ' + url)
    (cache / filename).write_bytes(content)
    manifest[url] = {'path': str(cache / filename), 'sha256': hashlib.sha256(content).hexdigest()}
(cache / 'manifest.json').write_text(json.dumps(manifest, indent=2) + '\n')
assert Path('/src/index.html').is_file()
print('Cached', len(manifest), 'external script/style resources; static sources are in /src.')
PY
