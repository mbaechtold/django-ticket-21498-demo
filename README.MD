# Django Ticket #21498 Example Project

This is a very simple Django project serving as an example for the Django 
ticket [#21498](https://code.djangoproject.com/ticket/21498) as requested in
[a comment of Claude Paroz](https://code.djangoproject.com/ticket/21498#comment:27).

The problem is that Django creates a new model migration if the fields of your
models happen to have an i18n verbose name and you change the language code of 
your Django project and

Let's see that step by step.

## Setup

Python 3.5 has been used, but Python 2.7 should work too. The bug has been 
reproduced with  Django 1.10.5 and Django 1.9.12.

So let's start by creating a new Django project:

    django-admin startproject example .
    
Leave all the generated code as is, then create a Django app:

    python manage.py startapp polls

## Start coding

Add some models as described in the [Django tutorial](https://docs.djangoproject.com/en/1.10/intro/tutorial02/#creating-models):

    class Question(models.Model):
        question_text = models.CharField(max_length=200)
        pub_date = models.DateTimeField('date published')
    
    
    class Choice(models.Model):
        question = models.ForeignKey(Question, on_delete=models.CASCADE)
        choice_text = models.CharField(max_length=200)
        votes = models.IntegerField(default=0)

Append the `polls` app config to `INSTALLED_APPS`:

    INSTALLED_APPS = [
        ...
        'polls.apps.PollsConfig',
    ]

Create the initial migration for our app:

    python manage.py makemigrations polls
    
Note how the field `pub_date` gets a verbose name (*date published*) in the migration:

    ...
    migrations.CreateModel(
            name='Question',
        fields=[
            ('id', models.AutoField(auto_created=True, primary_key=True, serialize=False, verbose_name='ID')),
            ('question_text', models.CharField(max_length=200)),
            ('pub_date', models.DateTimeField(verbose_name='date published')),
        ],
    ),
    ...
    
Configure `LOCALE_PATHS` and `LANGUAGES`:

    LOCALE_PATHS = [
        os.path.join(BASE_DIR, 'polls', 'locales'),
    ]

    LANGUAGES = (
        ('en', 'English'),
        ('de', 'German'),
    )

Please note that `LANGUAGE_CODE` still has the value from when the project has been generated:
    
    LANGUAGE_CODE = 'en-us'
    
Mark the field's verbose name for translation:

    from django.utils.translation import ugettext as _

    class Question(models.Model):
        ...
        pub_date = models.DateTimeField(_('date published'))
       
Generate the translation files for our `polls` app:

    cd polls
    python ../manage.py makemessages -l de -l en
    cd ..
    
Add the German translation (e.g. "Veröffentlichungsdatum")for the field's verbose name in the 
file `locales/de/LC_MESSAGES/django.po` and compile the translations:

    python manage.py compilemessages

Running `makemigrations` will (correctly) not detect any changes:

    python manage.py makemigrations                                                                                          ~/Developer/django-migration-bug  
    No changes detected
    
Until here, everything has been business as usual.

## Identifying the bug

Now change the language code to German:

    LANGUAGE_CODE = 'de'
    
Running `makemigrations` will generate a new migration altering the database field just
because its verbose name has changed:

    operations = [
        migrations.AlterField(
            model_name='question',
            name='pub_date',
            field=models.DateTimeField(verbose_name='Veröffentlichungsdatum'),
        ),
    ]

Changing the language code back to `en` and running `makemigrations` again will
create another migration.

Chaning the language code should definitely not cause new migrations to be created.
