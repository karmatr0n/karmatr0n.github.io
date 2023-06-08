---
layout: post
title:  "How to list all the installed apps in your iDevices"
date:   2015-01-30 22:30:00
tags: ios ruby forensics
---

I wrote a program using the _idevice rubygem_ to print a list of all applications installed on each device based on iOS, using the bindings for the _libimobiledevice and libplist libraries_, and running on _Mac OSX or Linux (Santoku or Remnux)_.

I think It could be useful for forensics or maybe just to identify how many apps you have installed on all your devices.

You have to connect all your devices and run the program to get a list including each UUID and name per device, with a list of installed applications that includes its category, version, name and path to the disk (nand).

The script does not depend on the jailbreak, and you should authorize your computer when each device asks if it should trust in that.

Its design is very simple, the program gets an array of strings with the UUID of each device connected to your computer through the _device_list_ method defined in the _iDevice_ class. So, behind the scenes the idevice
gem is using _Ruby-FFI_ to implement the bindings for the functions included in the libimobiledevice and libplist libraries.

Both libraries are implemented in C, the _libimobildevice_ library is designed to support the communication protocols by the USB ports connected to devices based on iOS (iPhone/iPod/iPad/AppleTV) using _libusbmuxd (linux) or usbmuxd (macosx)_. The _libplist_ library is designed to manipulate the Apple Property List format (binary or XML).

To get the name of each device the program uses the _device_name_ method in the _iDevice :: LockdownClient_ object. Then It will get the application list through the _browse_  method in the _iDevice::InstProxyClient_ object, and it prints the attributes defined in the _props_ array.

**Source code**
{% highlight ruby linenos %}
require 'bundler/setup'
require 'idevice'

def print_dev_info(idev)
  ldcli = Idevice::LockdownClient.attach(idev:idev)
  puts "%-40s    %s" % ["UUID", "Device name"]
  info = "%-40s => %s" % [idev.udid, ldcli.device_name]
  puts info
  puts "-" * info.length
end

def print_app_properties(cols)
  cols[2] = cols[2].to_s.slice(0,27) + "..." if cols[2].to_s.size > 30
  puts "%-15s %-15s %-30s %s/%s" % cols
end

Idevice.device_list.each do |udid|
  idev = Idevice::Idevice.attach(udid: udid)
  print_dev_info idev

  props = %w(ApplicationType CFBundleVersion CFBundleDisplayName Path
             CFBundleExecutable)
  print_app_properties props
  instpxy = Idevice::InstProxyClient.attach(idev: idev)
  instpxy.browse.sort { |a,b| a[props.at(0)] <=> b[props.at(0)] }.map do |app|
    print_app_properties app.values_at(*props)
  end

  puts ""
end
{% endhighlight %}

[Get the source code!](https://github.com/karmatr0n/apps-per-idev)

**Running the script**

{% highlight text %}
$ cd app-per-idev
$ bundle install
$ ./app-per-idev
{% endhighlight %}


**Output**

![Screenshot](/img/app-per-idev_screenshot.png)


**References**

  - [libmobiledevice](https://github.com/libimobiledevice/libimobiledevice)
  - [libplist](https://github.com/libimobiledevice/libplist)
  - [libusbmuxd](https://github.com/libimobiledevice/libusbmuxd)
  - [idevice](https://github.com/blueboxsecurity/idevice)
  - [ruby-ffI](https://github.com/ffi/ffi)
  - [ruby](https://ruby-lang.org)
