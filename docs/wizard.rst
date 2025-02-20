===========
Form wizard
===========

.. module:: formtools.wizard.views
    :synopsis: Splits forms across multiple Web pages.

The form wizard application splits :mod:`forms <django.forms>` across
multiple Web pages. It maintains state in one of the backends so that the
full server-side processing can be delayed until the submission of the final
form.

You might want to use this if you have a lengthy form that would be too
unwieldy for display on a single page. The first page might ask the user for
core information, the second page might ask for less important information,
etc.

The term "wizard", in this context, is `explained on Wikipedia`_.

.. _explained on Wikipedia: http://en.wikipedia.org/wiki/Wizard_%28software%29

How it works
============

Here's the basic workflow for how a user would use a wizard:

1. The user visits the first page of the wizard, fills in the form and
   submits it.
2. The server validates the data. If it's invalid, the form is displayed
   again, with error messages. If it's valid, the server saves the current
   state of the wizard in the backend and redirects to the next step.
3. Step 1 and 2 repeat, for every subsequent form in the wizard.
4. Once the user has submitted all the forms and all the data has been
   validated, the wizard processes the data -- saving it to the database,
   sending an email, or whatever the application needs to do.

Usage
=====

This application handles as much machinery for you as possible. Generally,
you just have to do these things:

1. Define a number of :class:`~django.forms.Form` classes -- one per
   wizard page.

2. Create a :class:`WizardView` subclass that specifies what to do once
   all of your forms have been submitted and validated. This also lets
   you override some of the wizard's behavior.

3. Create some templates that render the forms. You can define a single,
   generic template to handle every one of the forms, or you can define a
   specific template for each form.

4. Add ``formtools`` to your :setting:`INSTALLED_APPS` list in your settings
   file.

5. Point your URLconf at your :class:`WizardView` :meth:`~WizardView.as_view`
   method.

Defining ``Form`` classes
-------------------------

The first step in creating a form wizard is to create the
:class:`~django.forms.Form` classes.  These should be standard
:class:`django.forms.Form` classes, covered in the :mod:`forms documentation
<django.forms>`.  These classes can live anywhere in your codebase,
but convention is to put them in a file called :file:`forms.py` in your
application.

For example, let's write a "contact form" wizard, where the first page's form
collects the sender's email address and subject, and the second page collects
the message itself. Here's what the :file:`forms.py` might look like::

    from django import forms

    class ContactForm1(forms.Form):
        subject = forms.CharField(max_length=100)
        sender = forms.EmailField()

    class ContactForm2(forms.Form):
        message = forms.CharField(widget=forms.Textarea)


.. note::

    In order to use :class:`~django.forms.FileField` in any form, see the
    section :ref:`Handling files <wizard-files>` below to learn more about
    what to do.

Creating a ``WizardView`` subclass
----------------------------------

.. class:: SessionWizardView
.. class:: CookieWizardView

The next step is to create a :class:`formtools.wizard.views.WizardView`
subclass. You can also use the :class:`SessionWizardView` or
:class:`CookieWizardView` classes which preselect the backend used for
storing information during execution of the wizard (as their names indicate,
server-side sessions and browser cookies respectively).

.. note::

    To use the :class:`SessionWizardView` follow the instructions
    in the :mod:`sessions documentation <django.contrib.sessions>` on
    how to enable sessions.

We will use the :class:`SessionWizardView` in all examples but is completely
fine to use the :class:`CookieWizardView` instead. As with your
:class:`~django.forms.Form` classes, this :class:`WizardView` class can live
anywhere in your codebase, but convention is to put it in :file:`views.py`.

The only requirement on this subclass is that it implement a
:meth:`~WizardView.done()` method.

.. method:: WizardView.done(form_list, form_dict, **kwargs)

    This method specifies what should happen when the data for *every* form is
    submitted and validated. This method is passed a list and dictionary of
    validated :class:`~django.forms.Form` instances.

    In this simplistic example, rather than performing any database operation,
    the method simply renders a template of the validated data::

        from django.shortcuts import render
        from formtools.wizard.views import SessionWizardView

        class ContactWizard(SessionWizardView):
            def done(self, form_list, **kwargs):
                return render(self.request, 'done.html', {
                    'form_data': [form.cleaned_data for form in form_list],
                })

    Note that this method will be called via ``POST``, so it really ought to be a
    good Web citizen and redirect after processing the data. Here's another
    example::

        from django.http import HttpResponseRedirect
        from formtools.wizard.views import SessionWizardView

        class ContactWizard(SessionWizardView):
            def done(self, form_list, **kwargs):
                do_something_with_the_form_data(form_list)
                return HttpResponseRedirect('/page-to-redirect-to-when-done/')

    In addition to ``form_list``, the :meth:`~WizardView.done` method
    is passed a ``form_dict``, which allows you to access the wizard's
    forms based on their step names. This is especially useful when using
    :class:`NamedUrlWizardView`, for example::

        def done(self, form_list, form_dict, **kwargs):
            user = form_dict['user'].save()
            credit_card = form_dict['credit_card'].save()
            # ...

    .. versionchanged:: 1.7

        Previously, the ``form_dict`` argument wasn't passed to the
        ``done`` method.

See the section :ref:`Advanced WizardView methods <wizardview-advanced-methods>`
below to learn about more :class:`WizardView` hooks.

Creating templates for the forms
--------------------------------

Next, you'll need to create a template that renders the wizard's forms. By
default, every form uses a template called
:file:`formtools/wizard/wizard_form.html`. You can change this template name
by overriding either the
:attr:`~django.views.generic.base.TemplateResponseMixin.template_name` attribute
or the
:meth:`~django.views.generic.base.TemplateResponseMixin.get_template_names()`
method, which are documented in the
:class:`~django.views.generic.base.TemplateResponseMixin` documentation.  The
latter one allows you to use a different template for each form (:ref:`see the
example below <wizard-template-for-each-form>`).

This template expects a ``wizard`` object that has various items attached to
it:

* ``form`` -- The :class:`~django.forms.Form` or
  :class:`~django.forms.formsets.BaseFormSet` instance for the current step
  (either empty or with errors).

* ``steps`` -- A helper object to access the various steps related data:

  * ``step0`` -- The current step (zero-based).
  * ``step1`` -- The current step (one-based).
  * ``count`` -- The total number of steps.
  * ``first`` -- The first step.
  * ``last`` -- The last step.
  * ``current`` -- The current (or first) step.
  * ``next`` -- The next step.
  * ``prev`` -- The previous step.
  * ``index`` -- The index of the current step.
  * ``all`` -- A list of all steps of the wizard.

You can supply additional context variables by using the
:meth:`~WizardView.get_context_data` method of your :class:`WizardView`
subclass.

Here's a full example template:

.. code-block:: html+django

    {% extends "base.html" %}
    {% load i18n %}

    {% block head %}
    {{ wizard.form.media }}
    {% endblock %}

    {% block content %}
    <p>Step {{ wizard.steps.step1 }} of {{ wizard.steps.count }}</p>
    <form action="" method="post">{% csrf_token %}
    <table>
    {{ wizard.management_form }}
    {% if wizard.form.forms %}
        {{ wizard.form.management_form }}
        {% for form in wizard.form.forms %}
            {{ form.as_table }}
        {% endfor %}
    {% else %}
        {{ wizard.form }}
    {% endif %}
    </table>
    {% if wizard.steps.prev %}
    <button name="wizard_goto_step" type="submit" value="{{ wizard.steps.first }}">{% translate "first step" %}</button>
    <button name="wizard_goto_step" type="submit" value="{{ wizard.steps.prev }}">{% translate "prev step" %}</button>
    {% endif %}
    <input type="submit" value="{% translate "submit" %}"/>
    </form>
    {% endblock %}

.. note::

    Note that ``{{ wizard.management_form }}`` **must be used** for
    the wizard to work properly.

.. _wizard-urlconf:

Hooking the wizard into a URLconf
---------------------------------

.. method:: WizardView.as_view()

Finally, we need to specify which forms to use in the wizard, and then
deploy the new :class:`WizardView` object at a URL in the ``urls.py``. The
wizard's ``as_view()`` method takes a list of your
:class:`~django.forms.Form` classes as an argument during instantiation::

    from django.path import path

    from myapp.forms import ContactForm1, ContactForm2
    from myapp.views import ContactWizard

    urlpatterns = [
        path('contact/', ContactWizard.as_view([ContactForm1, ContactForm2])),
    ]

You can also pass the form list as a class attribute named ``form_list``::

    class ContactWizard(WizardView):
        form_list = [ContactForm1, ContactForm2]

.. _wizard-template-for-each-form:

Using a different template for each form
----------------------------------------

As mentioned above, you may specify a different template for each form.
Consider an example using a form wizard to implement a multi-step checkout
process for an online store. In the first step, the user specifies a billing
and shipping address. In the second step, the user chooses payment type. If
they chose to pay by credit card, they will enter credit card information in
the next step. In the final step, they will confirm the purchase.

Here's what the view code might look like::

    from django.http import HttpResponseRedirect
    from formtools.wizard.views import SessionWizardView

    FORMS = [("address", myapp.forms.AddressForm),
             ("paytype", myapp.forms.PaymentChoiceForm),
             ("cc", myapp.forms.CreditCardForm),
             ("confirmation", myapp.forms.OrderForm)]

    TEMPLATES = {"address": "checkout/billingaddress.html",
                 "paytype": "checkout/paymentmethod.html",
                 "cc": "checkout/creditcard.html",
                 "confirmation": "checkout/confirmation.html"}

    def pay_by_credit_card(wizard):
        """Return true if user opts to pay by credit card"""
        # Get cleaned data from payment step
        cleaned_data = wizard.get_cleaned_data_for_step('paytype') or {'method': 'none'}
        # Return true if the user selected credit card
        return cleaned_data['method'] == 'cc'


    class OrderWizard(SessionWizardView):
        def get_template_names(self):
            return [TEMPLATES[self.steps.current]]

        def done(self, form_list, **kwargs):
            do_something_with_the_form_data(form_list)
            return HttpResponseRedirect('/page-to-redirect-to-when-done/')
            ...

The ``urls.py`` file would contain something like::

    urlpatterns = [
        path('checkout/', OrderWizard.as_view(FORMS, condition_dict={'cc': pay_by_credit_card})),
    ]

The ``condition_dict`` can be passed as attribute for the ``as_view()``
method or as a class attribute named ``condition_dict``::

    class OrderWizard(WizardView):
        condition_dict = {'cc': pay_by_credit_card}

Note that the ``OrderWizard`` object is initialized with a list of pairs.
The first element in the pair is a string that corresponds to the name of the
step and the second is the form class.

In this example, the
:meth:`~django.views.generic.base.TemplateResponseMixin.get_template_names()`
method returns a list containing a single template, which is selected based on
the name of the current step.

.. _wizardview-advanced-methods:

Advanced ``WizardView`` methods
===============================

.. class:: WizardView

    Aside from the :meth:`~done()` method, :class:`WizardView` offers a few
    advanced method hooks that let you customize how your wizard works.

    Some of these methods take an argument ``step``, which is a zero-based
    counter as string representing the current step of the wizard. (E.g., the
    first form is ``'0'`` and the second form is ``'1'``)

.. method:: WizardView.get_form_prefix(step=None, form=None)

    Returns the prefix which will be used when calling the form for the given
    step. ``step`` contains the step name, ``form`` the form class which will
    be called with the returned prefix.

    If no ``step`` is given, it will be determined automatically. By default,
    this simply uses the step itself and the ``form`` parameter is not used.

    For more, see the :ref:`form prefix documentation <form-prefix>`.

.. method:: WizardView.get_form_initial(step)

    Returns a dictionary which will be passed as the
    :attr:`~django.forms.Form.initial` argument when instantiating the Form
    instance for step ``step``. If no initial data was provided while
    initializing the form wizard, an empty dictionary should be returned.

    The default implementation::

        def get_form_initial(self, step):
            return self.initial_dict.get(step, {})

.. method:: WizardView.get_form_kwargs(step)

    Returns a dictionary which will be used as the keyword arguments when
    instantiating the form instance on given ``step``.

    The default implementation::

        def get_form_kwargs(self, step):
            return {}

.. method:: WizardView.get_form_instance(step)

    This method will be called only if a :class:`~django.forms.ModelForm` is
    used as the form for step ``step``.

    Returns an :class:`~django.db.models.Model` object which will be passed as
    the ``instance`` argument when instantiating the ``ModelForm`` for step
    ``step``.  If no instance object was provided while initializing the form
    wizard, ``None`` will be returned.

    The default implementation::

        def get_form_instance(self, step):
            return self.instance_dict.get(step, None)

.. method:: WizardView.get_context_data(form, **kwargs)

    Returns the template context for a step. You can overwrite this method
    to add more data for all or some steps. This method returns a dictionary
    containing the rendered form step.

    The default template context variables are:

    * Any extra data the storage backend has stored
    * ``wizard`` -- a dictionary representation of the wizard instance with the
      following key/values:

      * ``form`` -- :class:`~django.forms.Form` or
        :class:`~django.forms.formsets.BaseFormSet` instance for the current step
      * ``steps`` -- A helper object to access the various steps related data
      * ``management_form`` -- all the management data for the current step

    Example to add extra variables for a specific step::

        def get_context_data(self, form, **kwargs):
            context = super().get_context_data(form=form, **kwargs)
            if self.steps.current == 'my_step_name':
                context.update({'another_var': True})
            return context

.. method:: WizardView.get_prefix(request, *args, **kwargs)

    This method returns a prefix for use by the storage backends. Backends use
    the prefix as a mechanism to allow data to be stored separately for each
    wizard. This allows wizards to store their data in a single backend
    without overwriting each other.

    You can change this method to make the wizard data prefix more unique to,
    e.g. have multiple instances of one wizard in one session.

    Default implementation::

        def get_prefix(self, request, *args, **kwargs):
            # use the lowercase underscore version of the class name
            return normalize_name(self.__class__.__name__)

    .. versionchanged:: 1.0

        The ``request`` parameter was added.

.. method:: WizardView.get_form(step=None, data=None, files=None)

    This method constructs the form for a given ``step``. If no ``step`` is
    defined, the current step will be determined automatically. If you override
    ``get_form``, however, you will need to set ``step`` yourself using
    ``self.steps.current`` as in the example below. The method gets three
    arguments:

    * ``step`` -- The step for which the form instance should be generated.
    * ``data`` -- Gets passed to the form's data argument
    * ``files`` -- Gets passed to the form's files argument

    You can override this method to add extra arguments to the form instance.

    Example code to add a user attribute to the form on step 2::

        def get_form(self, step=None, data=None, files=None):
            form = super().get_form(step, data, files)

            # determine the step if not given
            if step is None:
                step = self.steps.current

            if step == '1':
                form.user = self.request.user
            return form

.. method:: WizardView.process_step(form)

    Hook for modifying the wizard's internal state, given a fully validated
    :class:`~django.forms.Form` object. The Form is guaranteed to have clean,
    valid data.

    This method gives you a way to post-process the form data before the data
    gets stored within the storage backend. By default it just returns the
    ``form.data`` dictionary. You should not manipulate the data here but you
    can use it to do some extra work if needed (e.g. set storage extra data).

    Note that this method is called every time a page is rendered for *all*
    submitted steps.

    The default implementation::

        def process_step(self, form):
            return self.get_form_step_data(form)

.. method:: WizardView.process_step_files(form)

    This method gives you a way to post-process the form files before the
    files gets stored within the storage backend. By default it just returns
    the ``form.files`` dictionary. You should not manipulate the data here
    but you can use it to do some extra work if needed (e.g. set storage
    extra data).

    Default implementation::

        def process_step_files(self, form):
            return self.get_form_step_files(form)

.. method:: WizardView.render_goto_step(step, goto_step, **kwargs)

    This method is called when the step should be changed to something else
    than the next step. By default, this method just stores the requested
    step ``goto_step`` in the storage and then renders the new step.

    If you want to store the entered data of the current step before rendering
    the next step, you can overwrite this method.

.. method:: WizardView.render_revalidation_failure(step, form, **kwargs)

    When the wizard thinks all steps have passed it revalidates all forms with
    the data from the backend storage.

    If any of the forms don't validate correctly, this method gets called.
    This method expects two arguments, ``step`` and ``form``.

    The default implementation resets the current step to the first failing
    form and redirects the user to the invalid form.

    Default implementation::

        def render_revalidation_failure(self, step, form, **kwargs):
            self.storage.current_step = step
            return self.render(form, **kwargs)

.. method:: WizardView.get_form_step_data(form)

    This method fetches the data from the ``form`` Form instance and returns the
    dictionary. You can use this method to manipulate the values before the data
    gets stored in the storage backend.

    Default implementation::

        def get_form_step_data(self, form):
            return form.data

.. method:: WizardView.get_form_step_files(form)

    This method returns the form files. You can use this method to manipulate
    the files before the data gets stored in the storage backend.

    Default implementation::

        def get_form_step_files(self, form):
            return form.files

.. method:: WizardView.render(form, **kwargs)

    This method gets called after the GET or POST request has been handled. You
    can hook in this method to, e.g. change the type of HTTP response.

    Default implementation::

        def render(self, form=None, **kwargs):
            form = form or self.get_form()
            context = self.get_context_data(form=form, **kwargs)
            return self.render_to_response(context)

.. method:: WizardView.get_cleaned_data_for_step(step)

    This method returns the cleaned data for a given ``step``. Before returning
    the cleaned data, the stored values are revalidated through the form. If
    the data doesn't validate, ``None`` will be returned.

.. method:: WizardView.get_all_cleaned_data()

    This method returns a merged dictionary of all form steps' ``cleaned_data``
    dictionaries. If a step contains a ``FormSet``, the key will be prefixed
    with ``formset-`` and contain a list of the formset's ``cleaned_data``
    dictionaries. Note that if two or more steps have a field with the same
    name, the value for that field from the latest step will overwrite the
    value from any earlier steps.

Providing initial data for the forms
====================================

.. attribute:: WizardView.initial_dict

    Initial data for a wizard's :class:`~django.forms.Form` objects can be
    provided using the optional :attr:`~WizardView.initial_dict` keyword
    argument. This argument should be a dictionary mapping the steps to
    dictionaries containing the initial data for each step. The dictionary of
    initial data will be passed along to the constructor of the step's
    :class:`~django.forms.Form`::

        >>> from myapp.forms import ContactForm1, ContactForm2
        >>> from myapp.views import ContactWizard
        >>> initial = {
        ...     '0': {'subject': 'Hello', 'sender': 'user@example.com'},
        ...     '1': {'message': 'Hi there!'}
        ... }
        >>> # This example is illustrative only and isn't meant to be run in
        >>> # the shell since it requires an HttpRequest to pass to the view.
        >>> wiz = ContactWizard.as_view([ContactForm1, ContactForm2], initial_dict=initial)(request)
        >>> form1 = wiz.get_form('0')
        >>> form2 = wiz.get_form('1')
        >>> form1.initial
        {'sender': 'user@example.com', 'subject': 'Hello'}
        >>> form2.initial
        {'message': 'Hi there!'}

    The ``initial_dict`` can also take a list of dictionaries for a specific
    step if the step is a ``FormSet``.

    The ``initial_dict`` can also be added as a class attribute named
    ``initial_dict`` to avoid having the initial data in the ``urls.py``.

.. _wizard-files:

Handling files
==============

.. attribute:: WizardView.file_storage

To handle :class:`~django.forms.FileField` within any step form of the wizard,
you have to add a ``file_storage`` to your :class:`WizardView` subclass.

This storage will temporarily store the uploaded files for the wizard. The
``file_storage`` attribute should be a
:class:`~django.core.files.storage.Storage` subclass.

Django provides a built-in storage class (see :ref:`the built-in filesystem
storage class <builtin-fs-storage>`)::

    from django.conf import settings
    from django.core.files.storage import FileSystemStorage

    class CustomWizardView(WizardView):
        ...
        file_storage = FileSystemStorage(location=os.path.join(settings.MEDIA_ROOT, 'photos'))

.. warning::

    Please remember to take care of removing old temporary files, as the
    :class:`WizardView` will only remove these files if the wizard finishes
    correctly.

Conditionally view/skip specific steps
======================================

.. attribute:: WizardView.condition_dict

The :meth:`~WizardView.as_view` method accepts a ``condition_dict`` argument.
You can pass a dictionary of boolean values or callables. The key should match
the steps names (e.g. '0', '1').

If the value of a specific step is callable it will be called with the
:class:`WizardView` instance as the only argument. If the return value is true,
the step's form will be used.

This example provides a contact form including a condition. The condition is
used to show a message form only if a checkbox in the first step was checked.

The steps are defined in a ``forms.py`` file::

    from django import forms

    class ContactForm1(forms.Form):
        subject = forms.CharField(max_length=100)
        sender = forms.EmailField()
        leave_message = forms.BooleanField(required=False)

    class ContactForm2(forms.Form):
        message = forms.CharField(widget=forms.Textarea)

We define our wizard in a ``views.py``::

    from django.shortcuts import render
    from formtools.wizard.views import SessionWizardView

    def show_message_form_condition(wizard):
        # try to get the cleaned data of step 1
        cleaned_data = wizard.get_cleaned_data_for_step('0') or {}
        # check if the field ``leave_message`` was checked.
        return cleaned_data.get('leave_message', True)

    class ContactWizard(SessionWizardView):

        def done(self, form_list, **kwargs):
            return render(self.request, 'done.html', {
                'form_data': [form.cleaned_data for form in form_list],
            })

We need to add the ``ContactWizard`` to our ``urls.py`` file::

    from django.urls import path

    from myapp.forms import ContactForm1, ContactForm2
    from myapp.views import ContactWizard, show_message_form_condition

    contact_forms = [ContactForm1, ContactForm2]

    urlpatterns = [
        path('contact/', ContactWizard.as_view(contact_forms,
            condition_dict={'1': show_message_form_condition}
        )),
    ]

As you can see, we defined a ``show_message_form_condition`` next to our
:class:`WizardView` subclass and added a ``condition_dict`` argument to the
:meth:`~WizardView.as_view` method. The key refers to the second wizard step
(because of the zero based step index).

How to work with ModelForm and ModelFormSet
===========================================

.. attribute:: WizardView.instance_dict

WizardView supports :mod:`ModelForms <django.forms.models>` and
:ref:`ModelFormSets <model-formsets>`. Additionally to
:attr:`~WizardView.initial_dict`, the :meth:`~WizardView.as_view` method takes
an ``instance_dict`` argument that should contain model instances for steps
based on ``ModelForm`` and querysets for steps based on ``ModelFormSet``.

Usage of ``NamedUrlWizardView``
===============================

.. class:: NamedUrlWizardView
.. class:: NamedUrlSessionWizardView
.. class:: NamedUrlCookieWizardView

``NamedUrlWizardView`` is a :class:`WizardView` subclass which adds named-urls
support to the wizard. This allows you to have separate URLs for every step.
You can also use the :class:`NamedUrlSessionWizardView` or :class:`NamedUrlCookieWizardView`
classes which preselect the backend used for storing information (Django sessions and
browser cookies respectively).

To use the named URLs, you should not only use the :class:`NamedUrlWizardView` instead of
:class:`WizardView`, but you will also have to change your ``urls.py``.

The :meth:`~WizardView.as_view` method takes two additional arguments:

* a required ``url_name`` -- the name of the url (as provided in the ``urls.py``)
* an optional ``done_step_name`` -- the name of the done step, to be used in the URL

This is an example of a ``urls.py`` for a contact wizard with two steps, step 1 named
``contactdata`` and step 2 named ``leavemessage``::

    from django.urls import path, re_path

    from myapp.forms import ContactForm1, ContactForm2
    from myapp.views import ContactWizard

    named_contact_forms = (
        ('contactdata', ContactForm1),
        ('leavemessage', ContactForm2),
    )

    contact_wizard = ContactWizard.as_view(named_contact_forms,
        url_name='contact_step', done_step_name='finished')

    urlpatterns = [
        re_path(r'^contact/(?P<step>.+)/$', contact_wizard, name='contact_step'),
        path('contact/', contact_wizard, name='contact'),
    ]

Advanced ``NamedUrlWizardView`` methods
=======================================

.. method:: NamedUrlWizardView.get_step_url(step)

This method returns the URL for a specific step.

Default implementation::

    def get_step_url(self, step):
        return reverse(self.url_name, kwargs={'step': step})
