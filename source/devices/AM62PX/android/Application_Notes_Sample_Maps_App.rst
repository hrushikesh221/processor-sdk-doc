#######################################
Sample Maps Application for Car Variant
#######################################

As part of 10.0 SDK release, Android SDK now includes a pre-built maps
application created by Snapp Automotive. The app is developed on top of
the OSMDroid library and the source code of the app is available at
https://github.com/snappautomotive/SnappMaps.

.. note::

   For more information on this application, please visit the Snapp Automotive
   website at https://www.snappautomotive.io/


**********
Background
**********

When building the Car variant of AOSP there are no map applications
included and instead a green placeholder screen is seen. This application gets
included when building the ``userdebug`` builds to replace the green screen
placeholder.


*****************
Launching the app
*****************

   * Boot the EVM with binaries built using ``am62p_car-ap4a-userdebug`` lunch target

   * The SnappMaps application should be visible by default on the home screen


     .. Image:: /images/am62px_android_sample_maps_app.png
        :width: 600

.. note::

   Internet connection is required for this application to work.
