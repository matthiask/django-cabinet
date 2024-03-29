=========================================
django-cabinet - Media library for Django
=========================================

Version |release|

django-cabinet is a media library for Django implemented while trying to
write as little code as possible to keep maintenance at a minimum. At
the time of writing the projects consists of less than 1000 lines of
code (excluding tests), but still offers hierarchical folders,
downloads, images with primary point of interest (courtesy of
django-imagefield_) and drag-drop uploading of files directly
into the folder view.


Screenshots
===========

.. figure:: _static/root-folders.png

   List of root folders

.. figure:: _static/files-and-folders.png

   Folder view with subfolders and files

.. figure:: _static/image-ppoi.png

   File details with primary point of interest


Installation
============

- ``pip install django-cabinet``
- Add ``cabinet`` and ``imagefield`` to your ``INSTALLED_APPS``
- Maybe replace the file model by setting ``CABINET_FILE_MODEL``, but the
  default should be fine for most uses.


High-level overview
===================

django-cabinet comes with two concrete models, ``cabinet.File`` and
``cabinet.Folder``.

**Folders** can be nested into a hierarchy.

**Files** by default have two file fields, one for images based on
django-imagefield_ and one for downloads, a standard Django
``FileField``. Exactly one field has to be filled in. Files can only be
added to folders; files can never exist in the root folder.


Using cabinet files in your models
==================================

The recommended way to use cabinet files in your models is by using
``cabinet.fields.CabinetForeignKey``. When using ``raw_id_fields`` for
the file foreign key you automatically get an improved widget which 1.
allows directly uploading files inline without having to open the
``raw_id_fields`` popup and 2.  also displays small thumbnails for image
files.


Using django-cabinet as a CKEditor filebrowser
==============================================

django-cabinet has built-in support for being used as a `CKEditor 4 file
manager
<https://ckeditor.com/docs/ckeditor4/latest/guide/dev_file_browse_upload.html>`__.
Currently, it only supports browsing files or images. Directly uploading
files isn't supported since the file browser API does not know about
cabinet's folders and folders are required for adding files to cabinet.

The values for using django-cabinet as a file and image browser follow
(assuming you're using the default file model):

.. code-block:: javascript

    CKEDITOR.config.filebrowserBrowseUrl = "/admin/cabinet/file/?_popup=1";
    CKEDITOR.config.filebrowserImageUrl = "/admin/cabinet/file/?_popup=1&file_type=image_file";


Replacing the file model
========================

The file model is swappable, and should be easy to replace if you
require additional or different functionality. If, for example, you'd
want to move all PDFs to a different storage, build on the following
example code.

First, ``models.py``:

.. code-block:: python

    from django.db import models
    from django.db.models import signals
    from django.dispatch import receiver
    from django.utils.translation import gettext_lazy as _

    from cabinet.base import AbstractFile, ImageMixin, DownloadMixin

    class PDFMixin(models.Model):
        pdf_file = models.FileField(
            _('pdf'),
            upload_to=...,
            storage=...,
            blank=True,
        )

        class Meta:
            abstract = True
            verbose_name = _("PDF")
            verbose_name_plural = _("PDFs")

        # Cabinet requires a accept_file method on all mixins which
        # have a file field:
        def accept_file(self, value):
            if value.name.lower().endswith('.pdf'):
                self.pdf_file = value
                return True

    class File(AbstractFile, ImageMixin, PDFMixin, DownloadMixin):
        FILE_FIELDS = ['image_file', 'pdf_file', 'download_file']

        # Add caption and copyright, makes FileAdmin reuse easier.
        caption = models.CharField(
            _('caption'),
            max_length=1000,
            blank=True,
        )
        copyright = models.CharField(
            _('copyright'),
            max_length=1000,
            blank=True,
        )

        # Add additional fields if you want to.

    # If files should be automatically deleted (this is also the case when
    # using the default file model):
    @receiver(signals.post_delete, sender=File)
    def delete_files(sender, instance, **kwargs):
        instance.delete_files()


Next, ``admin.py``:

.. code-block:: python

    # You do not have to build on top of FileAdmin, but if there's no
    # good reason not to it's probably much less work. If not, you
    # still should take a long look at cabinet.base_admin.FileAdminBase

    from django.contrib import admin

    from cabinet.admin import FileAdmin

    from .models import File

    # The default should probably work fine.
    admin.site.register(File, FileAdmin)


Then, add ``CABINET_FILE_MODEL = 'yourapp.File'`` to your Django
settings and at last you're ready for running ``./manage.py
makemigrations`` and ``./manage.py migrate``.


.. include:: ../CHANGELOG.rst

.. _django-imagefield: https://django-imagefield.readthedocs.io/
