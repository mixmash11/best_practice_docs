# Django Best Practices

## Initialize Project

1. Run the cookiecutter image
    ```shell script
   cookiecutter https://github.com/mixmash11/cookiecutter-django-pycharm-docker
    ```
1. Initialize a Git repo in the directory
    ```shell script
   git init
   git add -A
   git commit -m "initial commit"
    ```
1. Link to a Git repo
    ```shell script
   git remote add origin https://github.com/mixmash11/{project_name}.git
   git branch -M master
   git push -u origin master
    ```
1. Run the local Docker compose (via Run)
1. Add the Docker Compose instance as an Interpreter
1. Import the Black file watcher xml file in pycharm, then change the field Program to: ```/home/mixmash11/.local/bin/black```
1. Run a migration (via Run)
1. Add a superuser
    ```shell script
   docker-compose -f local.yml run --rm django python manage.py createsuperuser
    ```

## New App

1. Create the application using the management command "startapp".
    ```shell script
    docker-compose -f local.yml run --rm django python manage.py startapp hotel_management
    ```
1. Fix permissions (via Run)
1. Move the application directory inside the project directory
1. Change the new app apps.py file
   ```python
   name = 'hotel_backend.hotel_management'
    ```
1. Add to local installed_apps list in settings (base.py)
    ```python
   LOCAL_APPS = [
        "hotel_backend.users.apps.UsersConfig",
        "hotel_backend.hotel_management.apps.HotelManagementConfig"
   ]
    ```
   
## New Model

1. Create the model
    ```python
    # models.py
    from django.db import models
    from django.urls import reverse
    from django_countries.fields import CountryField
    
    
    class ClubMember(models.Model):
        class Gender(models.TextChoices):
            FEMALE = "female", "Female"
            MALE = "male", "Male"
    
        membership_number = models.PositiveIntegerField(
            "Membership Number", primary_key=True
        )
        first_name = models.CharField("First Name", max_length=255)
        last_name = models.CharField("Last Name", max_length=255)
        gender = models.CharField("Gender", choices=Gender.choices, max_length=255)
        birthdate = models.DateField("Date of Birth")
        nationality = CountryField("Nationality")
        media_permission = models.BooleanField("Media Release", default=True)
    
        def __str__(self):
            return f"{self.membership_number} - {self.first_name} {self.last_name}"
    
        def get_absolute_url(self):
            return reverse("membership:detail", kwargs={"pk": self.pk})
    ```
1. Run migrations (via Run)
1. Create a factory for the model
    ```python
    # tests/factories.py
    import datetime
    
    import factory
    import factory.fuzzy
    
    from ..models import ClubMember
    
    
    class ClubMemberFactory(factory.django.DjangoModelFactory):
        membership_number = factory.Faker("random_int")
        first_name = factory.Faker("first_name")
        last_name = factory.Faker("last_name")
        gender = factory.fuzzy.FuzzyChoice([x[0] for x in ClubMember.Gender.choices])
        birthdate = factory.fuzzy.FuzzyDate(start_date=datetime.date(1990, 1, 1))
        nationality = factory.Faker("country_code")
    
        class Meta:
            model = ClubMember
    ```
1. Create a test for the string and get_absolute_url methods
    ```python
    # tests/test_models.py
    import pytest
    
    from ..models import ClubMember
    from .factories import ClubMemberFactory
    
    pytestmark = pytest.mark.django_db
    
    
    def test__str__():
        membership_number = 100
        first_name = "Sally"
        last_name = "Skier"
        cm_string = f"{membership_number} - {first_name} {last_name}"
    
        club_member = ClubMemberFactory(
            membership_number=membership_number, first_name=first_name, last_name=last_name
        )
    
        assert club_member.__str__() == cm_string
        assert str(club_member) == cm_string
    
    
    def test_get_absolute_url():
        member = ClubMemberFactory()
        url = member.get_absolute_url()
        assert url == f"/membership/{member.pk}/"
    
    ```
1. Run tests (via Run)
1. Register model in the admin
    ```python
    # admin.py
    from django.contrib import admin
    from .models import ClubMember
    
    admin.site.register(ClubMember)
    ```

## Making a ListView

1. Add the ListView to the Views file
    ```python
    # membership/views.py
    from django.views.generic import ListView, DetailView, CreateView, UpdateView
    
    from .models import ClubMember
    
    
    class ClubMemberListView(ListView):
        model = ClubMember
        ordering = ["-membership_number"]
    ```
   
## Adding the URLs file and linking it to the main project
   
1. Make and fill the URL File
    ```python
    # membership/urls.py
    from django.urls import path
    from . import views
    
    app_name = "membership"
    urlpatterns = [path(route="", view=views.ClubMemberListView.as_view(), name="list")]
    ```
1. Add the URL to the app to project level urls.py file
    ```python
   urlpatterns = [
   # ...
   # FSC Management Application
    path(
        "membership/",
        include("fsc_management_application.membership.urls", namespace="membership"),
    ),
   ]

    ```

## Adding a URL tag to an HTML Template
1. Add a tag using the "url" tag
    ```html
    <a class="nav-link" href="{% url 'membership:update' clubmember.membership_number %}">Membership</a>
    ```

## Adding a Template (to a ListView)
1. Create a directory with the app name in the {project}/templates/ directory
1. Create an HTML file named after the view in the directory (i.e. a ClubMember ListView becomes clubmember_list.html)
1. Create the template file
1. Create tests for the view
    ```python
    # tests/test_views.py
    import datetime
    
    import pytest
    from pytest_django.asserts import assertContains
    
    from django.urls import reverse
    from django.contrib.sessions.middleware import SessionMiddleware
    from django.test import RequestFactory
    
    from fsc_management_application.users.models import User
    
    from ..models import ClubMember
    from ..views import ClubMemberCreateView, ClubMemberListView, ClubMemberDetailView
    from .factories import ClubMemberFactory
    from ...users.tests.factories import UserFactory
    
    pytestmark = pytest.mark.django_db
    
    
    @pytest.fixture
    def user():
        return UserFactory()
    
    
    @pytest.fixture
    def member():
        return ClubMemberFactory()
    
    
    def test_clubmember_list_view(rf):
        request = rf.get(reverse("membership:list"))
        response = ClubMemberListView.as_view()(request)
        assertContains(response, "Member List")
    
    
    def test_clubmember_list_view_contains_2_members(rf):
        member1 = ClubMemberFactory()
        member2 = ClubMemberFactory()
        request = rf.get(reverse("membership:list"))
        response = ClubMemberListView.as_view()(request)
        assertContains(response, member1.membership_number)
        assertContains(response, member2.membership_number)
    ```
   
## Template for List View
```html
{# clubmember_list.html #}

{% extends "base.html" %}

{% block title %}Member List{% endblock %}

{% block content %}
  <div class="container-fluid">
    <div class="jumbotron">
      <h2>Member List</h2>
    </div>
    <div class="container border rounded p-4">
      <div class="table-responsive">
        <table class="table table-striped table-lg">
          <thead>
          <tr>
            <th scope="col">#</th>
            <th scope="col">First</th>
            <th scope="col">Last</th>
            <th scope="col"></th>
          </tr>
          </thead>
          {% for member in clubmember_list %}
            <tr>
              <th scope="row">{{ member.membership_number }}</th>
              <td>{{ member.first_name }}</td>
              <td>{{ member.last_name }}</td>
              <td>
                <a href="{% url 'membership:detail' member.membership_number %}" class="btn btn-info"
                   role="button">View</a>
              </td>
            </tr>
          {% endfor %}
        </table>
      </div>
      <div class="border rounded p-2">
        <p class="text-center">Need to add a new club member?</p>
        <p class="text-center">
          <a href="{% url 'membership:add' %}" class="btn btn-primary" role="button">Add Member</a>
        </p>
      </div>
    </div>

  </div>

{% endblock %}
```
   
   
## Adding a DetailView to a Model
1. Create the detail view in the views file
    ```python
    class TripDetailView(DetailView):
        model = Trip
    ```
1. Add url listing to app URLs file
    ```python
    urlpatterns = [
        # ...
        path(route="<slug:slug>/", view=views.TripDetailView.as_view(), name="detail"),
    ]
    ```
1. Add URL to links
    ```html
    <a href="{% url 'trips:detail' trip.slug %}">View</a>
    ```
1. Add template file named {model_name}_detail.html to template directory for app
1. Add tests for the view
    ```python
    # tests/test_views.py
    import datetime
    
    import pytest
    from pytest_django.asserts import assertContains
    
    from django.urls import reverse
    from django.contrib.sessions.middleware import SessionMiddleware
    from django.test import RequestFactory
    
    from fsc_management_application.users.models import User
    
    from ..models import ClubMember
    from ..views import ClubMemberCreateView, ClubMemberListView, ClubMemberDetailView
    from .factories import ClubMemberFactory
    from ...users.tests.factories import UserFactory
    
    pytestmark = pytest.mark.django_db
    
    
    @pytest.fixture
    def user():
        return UserFactory()
    
    
    @pytest.fixture
    def member():
        return ClubMemberFactory()
    
    
    def test_clubmember_detail_view(rf, member):
        url = reverse("membership:detail", kwargs={"pk": member.pk})
        request = rf.get(url)
        callable_obj = ClubMemberDetailView.as_view()
        response = callable_obj(request, pk=member.pk)
        assertContains(response, member.membership_number)
    ```

## Detail Page Example
```html
{# clubmember_detail.html #}
{% extends "base.html" %}

{% block title %}Club Member: {{ clubmember }}{% endblock title %}

{% block content %}
  <div class="container-fluid">
    <div class="jumbotron">
      <h2>{{ clubmember.membership_number }}</h2>
      <p class="lead">{{ clubmember.first_name }} {{ clubmember.last_name }}</p>
      <hr class="my-4">
      <a class="btn btn-primary btn-lg" href="{% url 'membership:update' clubmember.membership_number %}"
         role="button">Update</a>
    </div>
    <div class="container border rounded p-4">
      <h3>Member Information</h3>
      <hr class="my-4">
      <div class="table">
        <table class="table table-striped table-lg">
          <tbody>
          <tr>
            <td>Membership Number</td>
            <td>{{ clubmember.membership_number }}</td>
          </tr>
          <tr>
            <td>Name</td>
            <td>{{ clubmember.first_name }} {{ clubmember.last_name }}</td>
          </tr>
          <tr>
            <td>Birthdate</td>
            <td>{{ clubmember.birthdate }}</td>
          </tr>
          <tr>
            <td>Gender</td>
            <td>{{ clubmember.gender.capitalize }}</td>
          </tr>
          <tr>
            <td>Nationality</td>
            <td>{{ clubmember.nationality.name }} {{ clubmember.nationality.unicode_flag }}</td>
          </tr>
          <tr>
            <td>Media Release?</td>
            {% if clubmember.media_permission %}
              <td>
                <span class="btn btn-success">Yes</span>
              </td>
            {% else %}
              <td>
                <span class="btn btn-danger">No</span>
              </td>
            {% endif %}
          </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>

{% endblock content %}
```

## Add a CreateView and an UpdateView to a Model
1. Add a get_absolute_url method to the Model
    ```python
    from django.urls import reverse
    
    class ClubMember(models.Model):
        # ...
        def get_absolute_url(self):
            return reverse("membership:detail", kwargs={"pk": self.pk})
    ```
1. Add a test for the get_absolute_url to the model test
    ```python
    def test_get_absolute_url():
        member = ClubMemberFactory()
        url = member.get_absolute_url()
        assert url == f"/membership/{member.pk}/"
    ```
1. Add a CreateView to the Views file with the fields to add
    ```python
    from django.views.generic import CreateView, UpdateView
    # ...
    class ClubMemberCreateView(CreateView):
    model = ClubMember

    fields = [
        "membership_number",
        "first_name",
        "last_name",
        "gender",
        "birthdate",
        "nationality",
        "media_permission",
    ]
    ```
1. Add the CreateView to the URLs file. Put it **before** the DetailView
    ```python
    urlpatterns = [
        # ListView or other views
        path(route="add/", view=views.ClubMemberCreateView.as_view(), name="add"),
        path(route="<int:pk>", view=views.ClubMemberDetailView.as_view(), name="detail"),
    ]
    ```
1. Create a template for the CreateView
    ```html
    {# clubmember_form.html #}
    {% extends "base.html" %}
    {% load crispy_forms_tags %}
    
    {% block title %}{{ view.action|default:"Add" }} Club Member{% endblock title %}
    
    {% block content %}
      <div class="container-fluid">
        <div class="jumbotron">
          <h2>{{ view.action|default:"Add" }} Club Member</h2>
        </div>
        <div class="container border rounded p-4">
          <h3>Member Information</h3>
          <hr class="my-4">
          <form method="post" action=".">
            {% csrf_token %}
            {{ form|crispy }}
            <button type="submit" class="btn btn-primary">Save</button>
          </form>
        </div>
      </div>
    {% endblock content %}
    ```
1. Add a Create link to a template (preferably the ListView)
    ```html
    <div class="border rounded p-2">
        <p class="text-center">Need to add a new club member?</p>
        <p class="text-center">
          <a href="{% url 'membership:add' %}" class="btn btn-primary" role="button">Add Member</a>
        </p>
    </div>
    ```
1. Add UpdateView to the Views file
    ```python
    from django.views.generic import UpdateView
    
    # ...
    
    class ClubMemberUpdateView(UpdateView):
        model = ClubMember
    
        fields = [
            "first_name",
            "last_name",
            "gender",
            "birthdate",
            "nationality",
            "media_permission",
        ]
    ```
1. Update the URLs file with a path **after** the add views and **before** the detail views
    ```python
    urlpatterns = [
        # base and add paths
        path(
            route="<int:pk>/update/",
            view=views.ClubMemberUpdateView.as_view(),
            name="update",
        ),
        # detail views
    ]
    ```
1. Add an Update link to a template (preferably the ListView)
    ```html
    <a class="btn btn-primary btn-lg" href="{% url 'membership:update' clubmember.membership_number %}"
         role="button">Update</a>
    ```
1. Add tests for the view
    ```python
    # tests/test_views.py
    import datetime
    
    import pytest
    from pytest_django.asserts import assertContains
    
    from django.urls import reverse
    from django.contrib.sessions.middleware import SessionMiddleware
    from django.test import RequestFactory
    
    from fsc_management_application.users.models import User
    
    from ..models import ClubMember
    from ..views import ClubMemberCreateView, ClubMemberListView, ClubMemberDetailView
    from .factories import ClubMemberFactory
    from ...users.tests.factories import UserFactory
    
    pytestmark = pytest.mark.django_db
    
    
    @pytest.fixture
    def user():
        return UserFactory()
    
    
    @pytest.fixture
    def member():
        return ClubMemberFactory()
    
    def test_clubmember_create_view(client, user):
        client.force_login(user)
        url = reverse("membership:add")
        response = client.get(url)
        assert response.status_code == 200
    
    
    def test_clubmember_create_view_has_correct_title(client, user):
        client.force_login(user)
        url = reverse("membership:add")
        response = client.get(url)
        assertContains(response, "Add Club Member")
    
    
    def test_clubmember_create_form_valid(client, user):
        client.force_login(user)
        form_data = {
            "membership_number": 1,
            "first_name": "Sally",
            "last_name": "Skier",
            "gender": ClubMember.Gender.FEMALE,
            "birthdate": datetime.date.today(),
            "nationality": "DE",
        }
        url = reverse("membership:add")
        response = client.post(url, form_data)
    
        member = ClubMember.objects.get(membership_number=1)
    
        assert member.first_name == "Sally"
        assert member.last_name == "Skier"
        assert member.gender == ClubMember.Gender.FEMALE
    
    
    def test_clubmember_update_view_has_correct_title(client, user, member):
        client.force_login(user)
        url = reverse("membership:update", kwargs={"pk": member.pk})
        response = client.get(url)
        assertContains(response, "Update Club Member")
    
    
    def test_clubmember_update(client, user, member):
        client.force_login(user)
        test_first_name = "Test"
        form_data = {
            "first_name": test_first_name,
            "last_name": member.last_name,
            "gender": member.gender,
            "birthdate": member.birthdate,
            "nationality": member.nationality,
        }
        url = reverse("membership:update", kwargs={"pk": member.pk})
        response = client.post(url, form_data)
        member.refresh_from_db()
        assert member.first_name == test_first_name
    ```
   
   
## View Test Example
```python
# tests/test_views.py
import datetime

import pytest
from pytest_django.asserts import assertContains

from django.urls import reverse
from django.contrib.sessions.middleware import SessionMiddleware
from django.test import RequestFactory

from fsc_management_application.users.models import User

from ..models import ClubMember
from ..views import ClubMemberCreateView, ClubMemberListView, ClubMemberDetailView
from .factories import ClubMemberFactory
from ...users.tests.factories import UserFactory

pytestmark = pytest.mark.django_db


@pytest.fixture
def user():
    return UserFactory()


@pytest.fixture
def member():
    return ClubMemberFactory()


def test_clubmember_list_view(rf):
    request = rf.get(reverse("membership:list"))
    response = ClubMemberListView.as_view()(request)
    assertContains(response, "Member List")


def test_clubmember_list_view_contains_2_members(rf):
    member1 = ClubMemberFactory()
    member2 = ClubMemberFactory()
    request = rf.get(reverse("membership:list"))
    response = ClubMemberListView.as_view()(request)
    assertContains(response, member1.membership_number)
    assertContains(response, member2.membership_number)


def test_clubmember_detail_view(rf, member):
    url = reverse("membership:detail", kwargs={"pk": member.pk})
    request = rf.get(url)
    callable_obj = ClubMemberDetailView.as_view()
    response = callable_obj(request, pk=member.pk)
    assertContains(response, member.membership_number)


def test_clubmember_create_view(client, user):
    client.force_login(user)
    url = reverse("membership:add")
    response = client.get(url)
    assert response.status_code == 200


def test_clubmember_create_view_has_correct_title(client, user):
    client.force_login(user)
    url = reverse("membership:add")
    response = client.get(url)
    assertContains(response, "Add Club Member")


def test_clubmember_create_form_valid(client, user):
    client.force_login(user)
    form_data = {
        "membership_number": 1,
        "first_name": "Sally",
        "last_name": "Skier",
        "gender": ClubMember.Gender.FEMALE,
        "birthdate": datetime.date.today(),
        "nationality": "DE",
    }
    url = reverse("membership:add")
    response = client.post(url, form_data)

    member = ClubMember.objects.get(membership_number=1)

    assert member.first_name == "Sally"
    assert member.last_name == "Skier"
    assert member.gender == ClubMember.Gender.FEMALE


def test_clubmember_update_view_has_correct_title(client, user, member):
    client.force_login(user)
    url = reverse("membership:update", kwargs={"pk": member.pk})
    response = client.get(url)
    assertContains(response, "Update Club Member")


def test_clubmember_update(client, user, member):
    client.force_login(user)
    test_first_name = "Test"
    form_data = {
        "first_name": test_first_name,
        "last_name": member.last_name,
        "gender": member.gender,
        "birthdate": member.birthdate,
        "nationality": member.nationality,
    }
    url = reverse("membership:update", kwargs={"pk": member.pk})
    response = client.post(url, form_data)
    member.refresh_from_db()
    assert member.first_name == test_first_name
```

## Add Additional Validation to a ModelForm
1. 
