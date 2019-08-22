    ---
title: "Edit Route: Editing and Updating a Resource"
slug: editing-and-deleting-a-playlist
---

Now we're checking off the seven **Resourceful Routes**. But we're not done until all of them are complete.

| URL              | HTTP Verb | Action  |
|------------------|-----------|---------|
| /                | GET       | index   |
| /playlists/new     | GET       | new     |
| /playlists         | POST      | create  |
| /playlists/:id     | GET       | show    |
| /playlists/:id/edit     | GET       | edit    |
| /playlists/:id     | PUT/PATCH | update  |
| /playlists/:id     | DELETE    | Destroy |

Normally in a site like Rotten Potatoes, we would only want authors of playlists to have the permission to edit or delete a playlist. However, because we do not have authentication yet, we're just going to let anyone edit and delete playlists.

# Edit Link

We want people to be able to edit and update playlists, so let's again start from the user's perspective. Edit and Update are similar to New and Create. First we need a link to the edit route that renders the `playlists_edit` template, and then we submit that edit form to the update route which will redirect to the show action.

> [action]
>
> So let's make the edit link in `templates/playlists_show.html`:
>
```html
<!-- templates/playlists_show.html -->
{% extends 'base.html' %}
>
{% block content %}
<h1>{{ playlist.title }}</h1>
<h2>{{ playlist.movieTitle }}</h2>
<p>{{ playlist.description }}</p>
>
<p><a href='/playlists/{{ playlist._id }}/edit'>Edit</a></p>
{% endblock %}
```

Ok, now if we click that edit link, we'll see that the route is not found. So let's make our edit action. The edit action is like the show action because we look up the `playlist` by its `_id` in the url parameter, but then we render the information in a template as editable form elements.

> [action]
>
> Add an edit route in `app.js`:
>
```python
# app.py
...
@app.route('/playlists/<id>/edit')
def playlists_edit(playlist_id):
    """Show the edit form for a playlist."""
    playlist = playlists.find_one({'_id': ObjectId(playlist_id)})
    return render_template('playlists_edit.html', playlist=playlist)
```

And of course we'll need that `playlists_edit` template. This template is a bit weird for three reasons:

1. **PUT vs. POST** - Although our update action will be expecting a PUT HTTP action, HTML forms cannot take an action attribute of `PUT`. This makes no sense, but nevertheless we must find a sensible workaround to HTML's shortcomings. So what we'll do is add a hidden input field with `name='_method'` and `value='PUT'`, and then on the server we'll be able to tell if we intended to do a PUT action.
1. **`value=''`** - we are using the `value` html attribute to pass in the values of the playlist we are trying to edit.
1. **`<textarea>{{}}</textarea>`** - the `<textarea>` HTML tag does not have a `value` attribute, so its contents must go between its open and close tags.

> [action]
>
> Add a `templates/playlists_edit.html` template:
>
```html
<!-- templates/playlists_edit.html -->
{% extends 'base.html' %}
>
{% block content %}
<form method='POST' action='/playlists/{{playlist._id}}'>
  <input type='hidden' name='_method' value='PUT'/>
  <fieldset>
    <legend>Edit Playlist</legend>
    <!-- TITLE -->
    <p>
      <label for='playlist-title'>Title</label><br>
      <input id='playlist-title' type='text' name='title' value='{{playlist.title}}'/>
    </p>
>
    <!-- MOVIE TITLE -->
    <p>
      <label for='movie-title'>Movie Title</label><br>
      <input id='movie-title' type='text' name='movieTitle' value='{{playlist.movieTitle}}' />
    </p>
>
    <!-- DESCRIPTION -->
    <p>
      <label for='playlist-description'>Description</label><br>
      <textarea id='playlist-description' name='description' rows='10' />{{playlist.description}}</textarea>
    </p>
  </fieldset>
  <!-- BUTTON -->
  <p>
    <button type='submit'>Save Playlist</button>
  </p>
</form>
{% endblock %}
```

# Update Route

Remember that we need to make sure our POST request is processed as a PUT request. We can do this by checking the `_method` field in the `request.form`.

However, what should we do if the `_method` field is not set to 'PUT'? In that case, we should throw an error. We can use the [werkzeug.exceptions](https://werkzeug.palletsprojects.com/en/0.15.x/exceptions/) library to do so.

> [action]
>
> Add the following import line to the top of `app.py`:
```python
from werkzeug.exceptions import NotFound
```
>
> Add the update route to `app.py`:
>
```python
# app.py
...
@app.route('/playlists/<playlist_id>', methods=['POST'])
def playlists_update(playlist_id):
    """Submit an edited playlist."""
    if request.form.get('_method') == 'PUT':
        updated_playlist = {
            'title': request.form.get('title'),
            'movieTitle': request.form.get('movieTitle'),
            'description': request.form.get('description')
        }
        playlists.update_one(
            {'_id': ObjectId(playlist_id)},
            {'$set': updated_playlist})
        return redirect(url_for('playlists_show', playlist_id=playlist_id))
    else:
        raise NotFound()
```

# DRY Code & Sub Templates

Did you notice that the code of our `playlists_new` and `playlists_edit` have a lot of similarities? Pretty much everything inside the `form` tag is the same. Let's use a **Partial Template** to pull that code out into its own template.

> [action]
>
> First make a folder called `partials` inside the `templates` folder. Now in that `partials` folder create the `playlists_form.html`.
>
```html
<!-- templates/partials/playlists_form.html -->
>
<fieldset>
    <legend>{{ title }}</legend>
    <!-- TITLE -->
    <p>
        <label for='playlist-title'>Title</label><br>
        <input id='playlist-title' type='text' name='title' value='{{ playlist.title }}'/>
    </p>
>
    <!-- MOVIE TITLE -->
    <p>
        <label for='movie-title'>Movie Title</label><br>
        <input id='movie-title' type='text' name='movieTitle' value='{{ playlist.movieTitle }}' />
    </p>
>
    <!-- DESCRIPTION -->
    <p>
        <label for='playlist-description'>Description</label><br>
        <textarea id='playlist-description' name='description' rows='10' />{{ playlist.description }}</textarea>
    </p>
</fieldset>
```

And now we can use this partial to replace that information in both our new and edit templates.

> [action]
>
> Update `templates/playlists_new.html` and `templates/playlists_edit.html` to use the partial:
>
```html
<!-- templates/playlists_new.html -->
{% extends 'base.html' %}
>
{% block content %}
<form method='POST' action='/playlists'>
    {% include 'partials/playlists_form.html' %}
>
    <!-- BUTTON -->
    <p>
        <button type='submit'>Save Playlist</button>
    </p>
>
</form>
{% endblock %}
```
>
```html
<!-- templates/playlists_edit.html -->
{% extends 'base.html' %}
>
{% block content %}
<form method='POST' action='/playlists/{{playlist._id}}'>
    <input type='hidden' name='_method' value='PUT'/>
    {% include 'partials/playlists_form.html' %}
    <!-- BUTTON -->
    <p>
        <button type='submit'>Save Playlist</button>
    </p>
</form>
{% endblock %}
```

Finally, notice how we included a `{{ title }}` in `templates/partials/playlists_form.html`. We need to ensure that gets populated correctly based on whether a user is editing a playlist or creating a new one.

> [action]
>
> Update your `new` and `edit` routes in `app.py` to include the `title` parameter:
>
```python
# app.py
...
>
@app.route('/playlists/new')
def playlists_new():
    """Create a new playlist."""
    return render_template('playlists_new.html', playlist={}, title='New Playlist')
>
...
>
@app.route('/playlists/<playlist_id>/edit')
def playlists_edit(playlist_id):
    """Show the edit form for a playlist."""
    playlist = playlists.find_one({'_id': ObjectId(playlist_id)})
    return render_template('playlists_edit.html', playlist=playlist, title='Edit Playlist')
```

Triumph! DRY code. (Don't Repeat Yourself)

# Now Commit

> [action]
>
>
```bash
$ git add .
$ git commit -m 'Users can edit, update, and destroy playlists'
$ git push
```

# Stretch Challenge: "Cancel" buttons

> [challenge]
>
> Sometimes people might start making a resource and then want to cancel. Can you add a "Cancel" button next to the "Save Playlist" button? What will it do? Where will it link to?