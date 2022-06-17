# Forms in Django 4.0+

## Background

Until Django 4.0 forms were rendered by string concatenation. A number of 3rd party
packages allowed for forms to be rendered as templates e.g. floppy forms, crispy-forms.

Django 4.0 introduced the capability to render forms using the template engine.
This allowed the form template to be set per form class or per instance if a template 
name is provided when calling `render()`.

## How-to

Let's see an example of how a form may currently be rendered in a template and see
how I would use the new capabilites in Django 4.0.

I found the example below from one of Django's own projects. It may also be that the 
form is written out in full in multiple places increasing the maintenance burden, let us
also see if we simplify the template and reduce the need for repetition. 

``` html+django
  <form method="post" action="">{% csrf_token %}
    <dl>
      <dt><label for="id_title">Title: {% if form.title.errors %}<span class="error">{{ form.title.errors|join:", " }}</span>{% endif %}</label></dt>
      <dd>{{ form.title }}</dd>
      <dt><label for="id_language">Language:</label></dt>
      <dd>{{ form.language }}{% if form.language.errors %}<span class="error">&nbsp;&nbsp;{{ form.language.errors|join:", " }}</span>{% endif %}</dd>
      <dt><label for="id_version">Version:</label></dt>
      <dd>{{ form.version }}{% if form.version.errors %}<span class="error">&nbsp;&nbsp;{{ form.version.errors|join:", " }}</span>{% endif %}</dd>
      <dt><label for="id_tag_list">Tags: {% if form.tags.errors %}<span class="error">{{ form.tags.errors|join:", " }}</span>{% endif %} </label></dt>
      <dd>{{ form.tags }} <br />Use commas between tag names, and hyphens for multiple words, like "tag1, tag2, tag3-with-long-name"</dd>
      <dt><label for="id_code">Code: {% if form.code.errors %}<span class="error">{{ form.code.errors|join:", " }}</span>{% endif %}</label></dt>
      <dd>{{ form.code }}</dd>
      <dt><label for="id_description">Description: {% if form.description.errors %}<span class="error">{{ form.description.errors|join:", " }}</span>{% endif %}</label></dt>
      <dd>{{ form.description }}<br />
      You can use Markdown syntax here (see the sidebar), but <strong>raw HTML will be removed</strong>.</dd>
      <dt><button type="submit">Save</button></dt>
    </dl>
  </form>
```

### Move data to your model or form

Firstly I would look to see if there are any elements of the form which could
be moved to either the model or form definition. In this scenario I have added
the `help_text` to the Form definition. See 
[Overriding the default fields](https://docs.djangoproject.com/en/4.0/topics/forms/modelforms/#overriding-the-default-fields)
for more details.

``` diff
class SnippetForm(forms.ModelForm):
    title = forms.CharField(validators=[validate_non_whitespace_only_string])
    description = forms.CharField(
        validators=[validate_non_whitespace_only_string],
        widget=forms.Textarea,
+        help_text=SafeString("You can use Markdown syntax here (see the sidebar), but <strong>raw HTML will be removed</strong>."),
    )
    code = forms.CharField(validators=[validate_non_whitespace_only_string], widget=forms.Textarea)

    class Meta:
        model = Snippet
        exclude = ("author", "bookmark_count", "rating_score")
+        help_texts = {
+            'tags': 'Use commas between tag names, and hyphens for multiple words, like "tag1, tag2, tag3-with-long-name".',
        }
```

I have chosen to mark the `help_text` as a `SafeString` here to avoid needing 
to use the `|safe` filter in the template.

Once this is defined the template can be updated. 

``` diff
-      <dd>{{ form.tags }} <br />Use commas between tag names, and hyphens for multiple words, like "tag1, tag2, tag3-with-long-name"</dd>
+      <dd>{{ form.tags }} <br />{{ form.tags.help_text }}</dd>

-      You can use Markdown syntax here (see the sidebar), but <strong>raw HTML will be removed</strong>.</dd>
+      {{ form.description.help_text }}</dd>
```

### Form Template

I believe the gold standardard is that in your main template you should use `{{ form }}`
to render your form.

Django 4.0 introduced template based rendering for forms which allows us to do
just this! Let's create a new template for this form. 

Currently the form is written out with each field being defined individually.
That is, the template includes `form.version`, `form.code` and so on. In our
example the layout of each field is the same and so we can loop over the forms
fields and render them with the same logic.

Here's our template to render the form.

``` html+django
  <dl>
      {% for field in form %}
      <dt>{{ field.label_tag }}
      {% if field.errors %} <span class="error">{{ field.errors|join:", " }}</span>{% endif %}
      </dt>
      <dd>
          {{ field }}        
          {% if field.help_text %}<br />{{ field.help_text }}{% endif %}
      </dd>
      {% endfor %}
      <dt><button type="submit">Save</button></dt>
  </dl>
```

Notable features being used here

1. Using `for field in form` to loop over each field in the form.

2. [`field.label_tag`](https://docs.djangoproject.com/en/4.0/ref/forms/api/#django.forms.BoundField.label_tag)
   to render the `<label>` which is rendered with the `for` attribute to 
   associate it with the `<input>`.

3. [`field.errors`](https://docs.djangoproject.com/en/4.0/ref/forms/api/#django.forms.BoundField.errors) 
    to render the field's errors. One additional step here could be make use of 
    the error list also being rendered with the 
    [template engine](https://docs.djangoproject.com/en/4.0/ref/forms/api/#customizing-the-error-list-format)
    and provide a separate template for the errors.

4. `{{ field }}` to render the `<input>`. NOTE: this is really just the field's 
    widget rather than the whole field i.e. including labels, errors, help text.

5. [`{{ field.help_text }}`](https://docs.djangoproject.com/en/4.0/ref/forms/api/#django.forms.BoundField.help_text)
   for the field's help text which is available since we moved it to the form 
   definition as a preliminary step. 

### Use the template

To use this template to render the form the following changes are requried. 

1. Define the [`template_name`](https://docs.djangoproject.com/en/4.0/ref/forms/api/#template-name) 
   on the form definition. 

2. Set the ordering of the fields using [`field_order`](https://docs.djangoproject.com/en/4.0/ref/forms/api/#django.forms.Form.field_order) 
   on the form definition. Without this, the form will be rendered in the order which 
   the fields are defined in the model.

3. Use `{{ form }}` to use the template

Let's see the diff for these changes.

In the form:

``` diff

 class SnippetForm(forms.ModelForm):
+    template_name = "cab/snippet_form.html"
+    field_order = ["title", "language", "version", "tags", "code", "description"]

```

In the template:

``` diff
 
-    <dl>
-      <dt><label for="id_title">Title: {% if form.title.errors %}<span class="error">{{ form.title.errors|join:", " }}</span>{% endif %}</label></dt>
-      <dd>{{ form.title }}</dd>
-      <dt><label for="id_language">Language:</label></dt>
-      <dd>{{ form.language }}{% if form.language.errors %}<span class="error">&nbsp;&nbsp;{{ form.language.errors|join:", " }}</span>{% endif %}</dd>
-      <dt><label for="id_version">Version:</label></dt>
-      <dd>{{ form.version }}{% if form.version.errors %}<span class="error">&nbsp;&nbsp;{{ form.version.errors|join:", " }}</span>{% endif %}</dd>
-      <dt><label for="id_tag_list">Tags: {% if form.tags.errors %}<span class="error">{{ form.tags.errors|join:", " }}</span>{% endif %} </label></dt>
-      <dd>{{ form.tags }} <br />Use commas between tag names, and hyphens for multiple words, like "tag1, tag2, tag3-with-long-name"</dd>
-      <dt><label for="id_code">Code: {% if form.code.errors %}<span class="error">{{ form.code.errors|join:", " }}</span>{% endif %}</label></dt>
-      <dd>{{ form.code }}</dd>
-      <dt><label for="id_description">Description: {% if form.description.errors %}<span class="error">{{ form.description.errors|join:", " }}</span>{% endif %}</label></dt>
-      <dd>{{ form.description }}<br />
-      You can use Markdown syntax here (see the sidebar), but <strong>raw HTML will be removed</strong>.</dd>
-      <dt><button type="submit">Save</button></dt>
-    </dl>
+   {{ form }} 
```

With the template being associated with the form means that we can avoid repetition
by having the form logic in a single place and use it where needed by using `{{ form }}`

## What about the future?

Django 4.1 will bring several new features to form rendering. 

1. The template to render forms can now be defined at the project level by
   settings the template name on your [`FORM_RENDERER`](https://docs.djangoproject.com/en/4.1/ref/settings/#form-renderer). 

   This means that you could have a single template for all of your sites form
   and set this centrally without needing to set `template_name` on every form.

   In Django 4.0 this _could_ be achieved by overriding the default form
   template. 

2. There's a new `<div>` based form template which implements `<fieldset>` and
   `<legend>` to group related inputs (e.g. radios). Using this template is
   highly recomended over the current [`as_table`](https://docs.djangoproject.com/en/4.0/ref/forms/api/#as-ul), 
   [`as_p`](https://docs.djangoproject.com/en/4.0/ref/forms/api/#as-p) and 
   [`as_ul`](https://docs.djangoproject.com/en/4.0/ref/forms/api/#as-ul) templates.

3. In additon to the [`form.label_tag`](https://docs.djangoproject.com/en/4.0/ref/forms/api/#django.forms.BoundField.label_tag) 
   we used above there's a 
   [`form.legend_tag`](https://docs.djangoproject.com/en/4.1/ref/forms/api/#django.forms.BoundField.legend_tag) 
   to render the field's label in a `<legend>`

4. And more, see the [full change log!](https://docs.djangoproject.com/en/dev/releases/4.1/#forms)

## Anything else?

I've got more thougths on potential future developments to ease this further.
But that's for another day. I'm off to go enjoy the summer first :-)
