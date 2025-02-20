==============
Notes on Yocto
==============

Steps to build EVerest in a Yocto image
=======================================

#. Set up a Yocto development environment.

   .. TIP::
      The `Yocto Project Quick Build`_ guide is an excellent place to start.

#. Proceed with your usual Yocto development setup process, deciding on an
   image, BSP, and other layers suitable for your hardware.

   We will be using ``core-image-full-cmdline`` throughout this guide.

#. Download a version of the |meta-everest layer|_

   .. TIP::
      The EVerest project includes a |meta-everest layer|_ for baking EVerest
      and its dependencies into a Yocto image.

   -  Clone the ``meta-everest`` layer into your Yocto build
      environment.

   -  Switch to a branch that matches the Yocto release you're using.
      While drafting this guide, we are using a ``kirkstone`` release of
      Yocto and the ``release/kirkstone/2023.3.0-rc1`` branch of
      ``meta-everest``.

#. Add ``meta-everest`` to your build. This is typically done either by
   manually editing ``conf/bblayers.conf`` or using the
   ``bitbake-layers add-layer path/to/layer`` command. As always, refer
   to the `Yocto
   documentation <https://docs.yoctoproject.org/4.0.17/dev-manual/layers.html>`__
   for more details.

#. Configure EVerest in a custom layer The ``meta-everest`` layer does
   not make assumptions about your EVerest configuration. In order for
   the EVerest service to start in the built image, you must provide a
   EVerest configuration specifying which module instances should be
   spun up and how they are stitched together. The EVerest service
   defined in ``meta-everest`` will not start unless a configuration
   file is found in the file system at ``/etc/everest/config.yaml``.

   The easiest way to add an EVerest configuration and other
   customizations is by creating a new layer, defining a recipe in the
   layer that copies a set configuration file into the image, and adding
   the layer to your build layers. The creation of the layer and adding
   it to your build layers can be done manually or, for instance, by
   using the ``bitbake-layers create-layer`` and
   ``bitbake-layers add-layer`` commands. Going forward, this guide
   assumes that a new layer has been created named
   ``meta-everest-config`` that contains a recipe named
   ``install-config``.

   After creating a new layer with an ``install-config`` recipe, copy
   the EVerest configuration of your choosing into a file named
   ``recipes-install-config/install-config/config.yaml`` in the
   ``meta-everest-config`` layer. You can then have the recipe install
   this ``config.yaml`` file into the image's
   ``/etc/everest/config.yaml`` file as part of the ``do_install``
   action in the recipe's ``install-config_0.1.bb`` BitBake file. The
   following is an example recipe that accomplishes this assuming that
   you're (a) using the MIT license for your layer that is created by
   ``bitbake-layers add-layer`` and (b) in a development environment
   organized in the way specified in the `Yocto Project Quick Build`_
   guide.

   .. code::

      SUMMARY = "Provide an EVerest configuration"
      DESCRIPTION = "Copy an EVerest configuration into an image"
      PR = "r0"

      LICENSE = "MIT"
      LIC_FILES_CHKSUM = "file://${COREBASE}/meta/COPYING.MIT;md5=3da9cfbcb788c80a0384361b4de20420"

      FILES:${PN} += "${datadir}/everest/*"
      SRC_URI = "file://${THISDIR}/config.yaml"

      inherit allarch

      do_install() {
          install -d ${D}/etc/everest

          # TODO: Change the following source path to match your local build directory structure
          install -m 0755 ${WORKDIR}/workdir/poky/meta-everest-config/recipes-install-config/install-config/config.yaml \
                          ${D}/etc/everest/config.yaml
      }

   This can be easily modified to support other licenses and build
   directory structures by modifying the ``LICENSE`` and
   ``LIC_FILES_CHKSUM`` variables and the source path in the second
   ``install`` command in ``do_install``.

#. Verify EVerest and its dependencies are present in your build's
   BitBake layers

   At this point, it is worth pausing to verify that you have included
   both the ``meta-everest`` and ``meta-everest-config`` layers into
   your build by verifying they appear in the ``BBLAYERS`` variable in
   ``conf/bblayers.conf``.

   This is also a good time to add additional layers to ``BBLAYERS`` to
   ensure EVerest's dependencies will be baked into your image. Which
   dependencies are needed will again depend on your EVerest
   configuration. At a minimum, you will want to include OpenEmbedded
   tools, mbedTLS from OpenEmbedded's networking layer, and Python (a
   dependency of the latter).

   .. code::

      BBLAYERS ?= " \
        /workdir/poky/meta \
        /workdir/poky/meta-poky \
        /workdir/poky/meta-yocto-bsp \
        /workdir/poky/meta-everest \
        /workdir/poky/meta-openembedded/meta-oe \
        /workdir/poky/meta-openembedded/meta-python \
        /workdir/poky/meta-openembedded/meta-networking \
        /workdir/poky/meta-everest-config \
        "

   Additional layers may be required depending on your choice of base
   image, BSP, EVerest configuration, etc.

#. Configure ``systemd`` The ``meta-everest`` layer stands up EVerest in
   the image as a ``systemd`` service. Ensure ``systemd`` is enabled in
   your image by including it as a ``DISTRO_FEATURE`` in
   ``conf/local.conf``:
   .. code::

      DISTRO_FEATURES:append = " systemd"
      DISTRO_FEATURES_BACKFILL_CONSIDERED += "sysvinit"
      VIRTUAL-RUNTIME_init_manager = "systemd"
      VIRTUAL-RUNTIME_initscripts = "systemd-compat-units"

#. Add needed recipes to your build configuration Append the
   ``everest-core``, ``mosquitto``, and ``install-config`` recipes to
   ``IMAGE_INSTALL`` (in ``conf/local.conf``) along with other recipes
   needed to stand up EVerest and the modules you're using. Using the
   ``core-image-full-cmdline`` image, for instance, these should also
   include ``tzdata`` to ensure timezone support is baked into the
   image.

   .. code::

      IMAGE_INSTALL:append = "\
          tzdata \
          everest-core \
          mosquitto \
          install-config \
          "

.. _`Yocto Project Quick Build`: https://docs.yoctoproject.org/brief-yoctoprojectqs/index.html

.. |meta-everest layer| replace:: ``meta-everest`` layer
.. _meta-everest layer: https://github.com/EVerest/meta-everest


Integrating Third-Party Modules
===============================

Sometimes you want to use EVerest to control hardware that lacks a board support module in EVerest.
Sometimes you want to incorporate value-added services into your charger to distinguish yourself from the competition.
And sometimes you just want to abuse EVerest's flexibility and use the platform to power a digital picture frame.
Whatever your needs, you'll likely find yourself wanting to extend EVerest with custom modules at some point in your journey.

The :ref:`module concept <detail_module_concept>`, :ref:`how modules are configured <existing_modules>`, and :ref:`how to build a custom module <tutorial_create_modules_main>` are covered in other sections of the documentation, as is a real-world example of :ref:`processing bank card payments <bank_transaction>`.
If you still have questions regarding how to integrate custom EVerest modules into your Yocto image, then this is the section for you.

Assumptions
-----------
#. You have a module called ``my-module`` that you wish to build and install into a Yocto image as part of a larger EVerest deployment.
#. Thanks to a careful reading of the aforementioned module documentation and examples, your ``my-module`` module works as expected in an EVerest build using an EVerest configuration file called ``config.yaml``.

Option 1: Patching ``my-module`` into the ``meta-everest`` Layer
----------------------------------------------------------------
#. Adapt the above guide to create an EVerest Yocto image for use with your hardware. This should include a ``meta-install-config`` BitBake layer as described above, but using the ``config.yaml`` that uses ``my-module``.

..
    TODO: Record how to add my-module into the EVerest build by patching the meta-everest layer

..
    TODO: Add a second section showing how to use a CMakeLists.txt akin to the one in tutorial_create_modules_main in order to build EVerest in a recipe from a new layer in a way that mirrors that tutorial.
