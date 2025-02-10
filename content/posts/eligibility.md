+++
date = '2025-02-09T20:04:23-06:00'
draft = false
title = 'Getting to the *core* of the eligibility system on *OS'
+++

# Getting to the *core* of the eligibility system on *OS
I have been really interested in Apple internals recently, and I have been really into debugging/reverse engineering. I got the tools from Apple’s open source `dyld` project building (there were many missing components), and decided to play around with it a bit. I extracted the shared cache, and also got a map of all of the symbols and things like that.

Each executable, or ‘file’ inside the shared cache is called an image, and every image has “fix-ups” Apple applies to it before adding it to the shared cache (or after). Originally, the entire purpose of the DYLD shared cache was performance, and it probably partially still is, but in my personal opinion, there is an aspect of secrecy to bundling every library on the system into one file.

The fix-ups that are applied to the binaries are incredibly hard to work around. For example, some references into other parts of the shared cache are transformed into static addresses in memory, and since the shared cache is always loaded in the same place, these static references can be made. Once extracted though, these references stay as constant memory addresses, and I am too lazy to modify the extraction code to remove the direct references to memory, so I stick to parsing the map file and determining what image the address resides in.

Once every image in the cache was extracted, I just started looking around. One thing I found interesting was the `/usr/lib/system` directory. There,  some libraries with private functionality exist.

At first, `libsystem_featureflags.dylib` piqued my interest. Of course I want to enable features that are unreleased as of now! But when I looked further into it, using static analysis tools, it didn’t seem too interesting.

Right alongside it, though, sat a seemingly innocuous `libsystem_eligibility.dylib`. I knew from the initial launch of Apple Intelligence in the betas that people did all sorts of things to the eligibility system to enable the features early, so I thought that I might give looking into it a shot.

In my static analysis tool of choice, I noticed that there were barely any functions defined, which would definitely make it easier for me to analyze. A few functions managed XPC messages and responses.

Right off the bat, I checked the XPC related functionality, and I found that the XPC messages were being sent to the mach service `com.apple.eligibilityd`. From there, I looked at the other functions, notably `_os_eligibility_get_internal_state` and `_os_eligibility_get_state_dump`, simply because they seemed interesting. Both of the functions used the XPC related utilities to send an object to eligibilityd, the only common pattern between all of them being that each message has a unique value for the `eligibility_message_type` property:

- `_os_eligibility_get_internal_state`: 4
- `_os_eligibility_get_state_dump`: 7

To try calling these methods, or manually sending the messages, I tried writing a C program that links to `libsystem_eligibility.dylib`, however, the linker wouldn’t let me. After researching this and consulting ChatGPT, I found that the best solution for my hacked-together demo I should use `dlopen` and `dlsym` to resolve the library and symbols, though there is probably a better way to do this.

```c
#include <dlfcn.h>
#include <stdio.h>
#include <stdlib.h>
#include <xpc/xpc.h>

typedef int (*os_eligibility_get_internal_state_t)(xpc_object_t *out_state);
typedef int (*os_eligibility_get_state_dump_t)(xpc_object_t *out_dump);

void print_xpc_object(xpc_object_t obj);

int main() {
  void *handle = NULL;
  os_eligibility_get_internal_state_t os_eligibility_get_internal_state = NULL;
  os_eligibility_get_state_dump_t os_eligibility_get_state_dump = NULL;
  xpc_object_t internal_state = NULL;
  xpc_object_t state_dump = NULL;
  int result = 0;

  handle = dlopen("/usr/lib/system/libsystem_eligibility.dylib", RTLD_LAZY);
  if (!handle) {
    fprintf(stderr, "Failed to open libsystem_eligibility.dylib: %s\n",
            dlerror());
    return 1;
  }

  dlerror();

  *(void **)(&os_eligibility_get_internal_state) =
      dlsym(handle, "os_eligibility_get_internal_state");
  char *error = dlerror();
  if (error != NULL) {
    fprintf(stderr, "Failed to find os_eligibility_get_internal_state: %s\n",
            error);
    dlclose(handle);
    return 1;
  }

  *(void **)(&os_eligibility_get_state_dump) =
      dlsym(handle, "os_eligibility_get_state_dump");
  error = dlerror();
  if (error != NULL) {
    fprintf(stderr, "Failed to find os_eligibility_get_state_dump: %s\n",
            error);
    dlclose(handle);
    return 1;
  }

  result = os_eligibility_get_internal_state(&internal_state);
  if (result != 0) {
    fprintf(stderr, "os_eligibility_get_internal_state failed with error: %d\n",
            result);
  } else {
    if (internal_state) {
      printf("internal state:\n");
      print_xpc_object(internal_state);
      xpc_release(internal_state);
    } else {
      printf("No internal state returned.\n");
    }
  }

  result = os_eligibility_get_state_dump(&state_dump);
  if (result != 0) {
    fprintf(stderr, "os_eligibility_get_state_dump failed with error: %d\n",
            result);
    dlclose(handle);
    return 1;
  }

  if (state_dump) {
    printf("state:\n");
    print_xpc_object(state_dump);
    xpc_release(state_dump);
  } else {
    printf("No state dump returned.\n");
  }

  dlclose(handle);
  return 0;
}

void print_xpc_object(xpc_object_t obj) {
  if (!obj) {
    printf("NULL XPC object.\n");
    return;
	}

	char *description = xpc_copy_description(obj);
	if (description) {
		printf("%s\n", description);
    free(description);
	} else {
    printf("Failed to get XPC object description.\n");
	}
}
```

> One small note, to get the program to build, I had to resolve to Xcode as it was the only way I could link with libxpc. If anyone knows another way to do so, please let me know. If you would like to build this yourself, go to your target in Xcode, and under General > Frameworks and Libraries click the plus button and search `libxpc`. You should see something like `libxpc.tbd`, select it and click okay, and you are all set to build.[^]

Then, I encountered a small problem:
```
eligibility_xpc_send_message_with_reply: Error returned trying to send xpc message 4: Connection interrupted
os_eligibility_get_internal_state failed with error: 54
```

Looking it up in the manpages (`errno`), code 54 means: “54 ECONNRESET Connection reset by peer. A connection was forcibly closed by a peer.” Evidently, eligibilityd is enforcing some restriction upon who can send it messages via XPC. I verified using the `dipc` tool that eligibilityd was truly receiving the request, and indeed it was. So, I resorted to my good ol’ static analysis tools to finish the job by finding out what I need to do to get an actual response to my messages.

In the entry point of `eligibilityd`, various XPC handlers were being setup, and one such handler held a reference to a subroutine. I figured this ought to be the thing I am looking for, but upon further inspection, it seemed to be modifying some other handler with another subroutine. In this one I could already see strings that looked like entitlements being passed into a function whose result determines the outcome of conditional branching. Upon inspection of the function, I confirmed that it checked entitlements. Now I have a list of every entitlement that I need to get past the checks.

```plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
"http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.private.eligibilityd.setInput</key>
    <true/>
    <key>com.apple.private.eligibilityd.resetDomain</key>
    <true/>
    <key>com.apple.private.eligibilityd.forceDomain</key>
    <true/>
    <key>com.apple.private.eligibilityd.internalState</key>
    <true/>
    <key>com.apple.private.eligibilityd.resetAllDomains</key>
    <true/>
    <key>com.apple.private.eligibilityd.forceDomainSet</key>
    <true/>
    <key>com.apple.private.eligibilityd.stateDump</key>
    <true/>
    <key>com.apple.private.eligibilityd.dumpSysdiagnoseDataToDir</key>
    <true/>
    <key>com.apple.private.eligibilityd.setTestMode</key>
    <true/>
    <key>com.apple.private.eligibilityd.testMode</key>
    <true/>
</dict>
</plist>
```

After setting Xcode to add these entitlements to my program and running it, I got these results (removed personal/identifying info):
```
Internal State:
<dictionary: ...> { count = 1, transaction: 0, voucher = 0x0, contents =
"OS_ELIGIBILITY_INTERNAL_STATE_GRACE_PERIOD_IN_EFFECT" =>
}

State Dump:
<dictionary: ...> { count = 5, transaction: 0, voucher = 0x0, contents =
"OS_ELIGIBILITY_STATE_DUMP_OVERRIDES" => ""
"OS_ELIGIBILITY_STATE_DUMP_INPUTS" => <dictionary: ...> { count = 12, transaction: 0, voucher = 0x0, contents =
"OS_ELIGIBILITY_INPUT_DEVICE_REGION_CODE" => "[DeviceRegionCodeInput deviceRegionCode:  ]"
"OS_ELIGIBILITY_INPUT_SHARED_IPAD" => "[SharediPadInput isSharediPad:  ]"
"OS_ELIGIBILITY_INPUT_GENERATIVE_MODEL_SYSTEM" => "[GenerativeModelSystemInput supportsGenerativeModelSystems:  ]"
"OS_ELIGIBILITY_INPUT_SIRI_LANGUAGE" => "[SiriLanguageInput language:  ]"
"OS_ELIGIBILITY_INPUT_EXTERNAL_BOOT_DRIVE" => "[ExternalBootDriveInput hasExternalBootDrive:  ]"
"OS_ELIGIBILITY_INPUT_COUNTRY_BILLING" => "[CountryBillingInput countryCode:  ]"
"OS_ELIGIBILITY_INPUT_COUNTRY_LOCATION" => "[LocatedCountryInput countryCodes:  ]"
"OS_ELIGIBILITY_INPUT_DEVICE_LANGUAGE" => "[DeviceLanguageInput deviceLanguages:  ]"
"OS_ELIGIBILITY_INPUT_DEVICE_CLASS" => "[DeviceClassInput deviceClass:  ]"
"OS_ELIGIBILITY_INPUT_DEVICE_LOCALE" => "[DeviceLocaleInput deviceLocale:  ]"
"OS_ELIGIBILITY_INPUT_GREYMATTER_ON_QUEUE" => "[GreymatterQueueInput onQueue:  ]"
"OS_ELIGIBILITY_INPUT_CHINA_CELLULAR" => "[ChinaCellularInput chinaCellularDevice:  ]"
}
"OS_ELIGIBILITY_STATE_DUMP_MOBILE_ASSET" => ""
"OS_ELIGIBILITY_STATE_DUMP_GLOBAL_STATE" => ""
"OS_ELIGIBILITY_STATE_DUMP_DOMAINS" => <dictionary: ...> { count = 39, transaction: 0, voucher = 0x0, contents =
"OS_ELIGIBILITY_DOMAIN_XCODE_LLM" => "..."
"OS_ELIGIBILITY_DOMAIN_SCANDIUM" => "..."
"OS_ELIGIBILITY_DOMAIN_KRYPTON" => "..."
"OS_ELIGIBILITY_DOMAIN_NICKEL" => "..."
"OS_ELIGIBILITY_DOMAIN_ARGON" => "..."
"OS_ELIGIBILITY_DOMAIN_LOTX" => "..."
"OS_ELIGIBILITY_DOMAIN_SODIUM" => "..."
"OS_ELIGIBILITY_DOMAIN_BORON" => "..."
"OS_ELIGIBILITY_DOMAIN_CARBON" => "..."
"OS_ELIGIBILITY_DOMAIN_PODCASTS_TRANSCRIPTS" => "..."
"OS_ELIGIBILITY_DOMAIN_NEON" => "..."
"OS_ELIGIBILITY_DOMAIN_CHLORINE" => "..."
"OS_ELIGIBILITY_DOMAIN_PHOSPHORUS" => "..."
"OS_ELIGIBILITY_DOMAIN_SEARCH_MARKETPLACES" => "..."
"OS_ELIGIBILITY_DOMAIN_STRONTIUM" => "..."
"OS_ELIGIBILITY_DOMAIN_IRON" => "..."
"OS_ELIGIBILITY_DOMAIN_HELIUM" => "..."
"OS_ELIGIBILITY_DOMAIN_ZINC" => "..."
"OS_ELIGIBILITY_DOMAIN_TITANIUM" => "..."
"OS_ELIGIBILITY_DOMAIN_CALCIUM" => "..."
"OS_ELIGIBILITY_DOMAIN_CHROMIUM" => "..."
"OS_ELIGIBILITY_DOMAIN_COBALT" => "..."
"OS_ELIGIBILITY_DOMAIN_MANGANESE" => "..."
"OS_ELIGIBILITY_DOMAIN_SWIFT_ASSIST" => "..."
"OS_ELIGIBILITY_DOMAIN_NITROGEN" => "..."
"OS_ELIGIBILITY_DOMAIN_OXYGEN" => "..."
"OS_ELIGIBILITY_DOMAIN_GREYMATTER" => "..."
"OS_ELIGIBILITY_DOMAIN_HYDROGEN" => "..."
"OS_ELIGIBILITY_DOMAIN_FLUORINE" => "..."
"OS_ELIGIBILITY_DOMAIN_BERYLLIUM" => "..."
"OS_ELIGIBILITY_DOMAIN_SULFUR" => "..."
"OS_ELIGIBILITY_DOMAIN_VANADIUM" => "..."
"OS_ELIGIBILITY_DOMAIN_SILICON" => "..."
"OS_ELIGIBILITY_DOMAIN_LITHIUM" => "..."
"OS_ELIGIBILITY_DOMAIN_MAGNESIUM" => "..."
"OS_ELIGIBILITY_DOMAIN_POTASSIUM" => "..."
"OS_ELIGIBILITY_DOMAIN_ALUMINUM" => "..."
"OS_ELIGIBILITY_DOMAIN_COPPER" => "..."
"OS_ELIGIBILITY_DOMAIN_RUBIDIUM" => "..."
}
}
Program ended with exit code: 0
```
This output provides some insight into how the system works, but I’d love to go more in depth into this.
