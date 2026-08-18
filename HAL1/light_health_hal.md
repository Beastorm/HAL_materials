# `LightHealth.h` and `LightHealth.cpp` — Line-by-Line Explanation

> **Filename note:** the project contains `LightHealth.h` and `LightHealth.cpp`. There is no `LightHealth.h.cpp`, so this document treats `LightHealth.h.cpp` in the request as `LightHealth.cpp`.
>
> Line numbers below match the current workspace files. Blank lines only separate logical sections and have no compile-time or runtime effect, so they are identified as separators rather than repeated individually.

## Document map

```text
PART I   Source explanation
         1-4   LightHealth.h/.cpp line-by-line and runtime flow

PART II  Generated AIDL and naming
         5-8   Generated files, include paths, .so output, Bn/Bp roles
         9-11  C++ namespace, Binder descriptor, and /default instance

PART III Android service architecture
         12-13 Binder service vs HAL and runtime process residence

PART IV  Native backend and service startup
         14-15 Why NDK is used, backend selection, and API stability
         16    service.cpp line-by-line startup behavior

PART V   Object lifetime
         17    SharedRefBase and Binder shared-reference ownership

PART VI  Binder calls, listeners, and threading
         18-20 ScopedAStatus, listener lifecycle, and death monitoring
         21-23 Binder pool vs worker, mutex purpose, and pool sizing
         24-26 Sync/oneway behavior, client pools, and object residence
```

## 1. Relationship between the two files

```text
 LightHealth.h                              LightHealth.cpp
 +----------------------------------+       +--------------------------------------+
 | Declares the LightHealth class   |       | Defines every declared method        |
 |                                  |       |                                      |
 | - AIDL methods                   |<------| - starts/stops worker                 |
 | - worker state                   |       | - runs core health check             |
 | - listener state                 |       | - registers/notifies listeners       |
 | - death-recipient callback       |       | - removes dead listeners             |
 +----------------+-----------------+       +-------------------+------------------+
                  |                                             |
                  v                                             v
       BnLightHealth generated base                  LightHealthDeviceIo + core
                  |                                             |
                  +---------------------+-----------------------+
                                        |
                                        v
                      ILightHealth/default Binder service
```

---

# Part A — `LightHealth.h`

## Lines 1–13: header protection and dependencies

- **Line 1 — `#pragma once`**: Tells the compiler to include this header only once per translation unit. It prevents duplicate class/type definitions if several headers include `LightHealth.h`.
- **Line 2 — blank separator**: Separates the header guard directive from includes.
- **Line 3 — `#include <.../BnLightHealth.h>`**: Imports the generated NDK Binder server base class. `LightHealth` must inherit from this class to implement the `ILightHealth` AIDL interface.
- **Line 4 — `#include <.../ILightHealthListener.h>`**: Imports the generated listener interface used for callbacks to clients.
- **Line 5 — `#include <.../LightHealthStatus.h>`**: Imports the AIDL-generated `UNKNOWN`, `HEALTHY`, and `TAMPERED` status enum.
- **Line 6 — `#include <android/binder_auto_utils.h>`**: Imports NDK Binder RAII utilities, including `ndk::ScopedAStatus` and `ndk::ScopedAIBinder_DeathRecipient`.
- **Line 7 — blank separator**: Separates Android/AIDL headers from C++ standard-library headers.
- **Line 8 — `#include <condition_variable>`**: Provides `std::condition_variable`, used to put the worker to sleep until a health check is requested.
- **Line 9 — `#include <functional>`**: Provides `std::function`, used by the injectable `CheckFunction` test hook.
- **Line 10 — `#include <memory>`**: Provides `std::shared_ptr`, `std::unique_ptr`, and related ownership utilities.
- **Line 11 — `#include <mutex>`**: Provides `std::mutex` and locking helpers that protect worker and listener state.
- **Line 12 — `#include <thread>`**: Provides `std::thread`, used for the dedicated health-check worker.
- **Line 13 — `#include <vector>`**: Provides the dynamic arrays holding listeners and death-recipient cookies.

## Lines 15–24: namespace, class, and type aliases

- **Line 14 — blank separator**: Ends the include section.
- **Line 15 — `namespace aidl::vendor::samsung::microxr::lighthealth {`**: Opens the implementation namespace. The implementation is Samsung-namespaced even though the inherited AIDL interface is in the Google vendor namespace.
- **Line 16 — blank separator**: Visually separates the namespace from the class declaration.
- **Line 17 — `class LightHealth final`**: Begins the service class declaration. `final` prevents another class from deriving from `LightHealth`.
- **Line 18 — `: public ...::BnLightHealth {`**: Publicly inherits from the generated Binder-native server class. This makes a `LightHealth` object publishable as an `ILightHealth` Binder service.
- **Line 19 — `public:`**: Begins the public interface visible to the service launcher, Binder framework, and tests.
- **Line 20 — `using ILightHealthListener =`**: Starts a short type alias declaration so the full generated namespace does not need to be repeated throughout the class.
- **Line 21 — full `ILightHealthListener` name**: Completes the alias. Inside this class, `ILightHealthListener` now means `aidl::vendor::google::microxr::lighthealth::ILightHealthListener`.
- **Line 22 — `using LightHealthStatus =`**: Starts a local alias for the generated AIDL status enum.
- **Line 23 — full `LightHealthStatus` name**: Completes the status alias.
- **Line 24 — `using CheckFunction = std::function<LightHealthStatus()>;`**: Defines a callable type that accepts no arguments and returns an AIDL status. Tests can inject this instead of running actual hardware I/O.

## Lines 26–33: constructor, destructor, and AIDL methods

- **Line 25 — blank separator**: Separates aliases from callable methods.
- **Line 26 — `explicit LightHealth(CheckFunction check_function = {});`**: Declares the constructor. The argument defaults to an empty function, meaning production hardware flow is used. `explicit` prevents accidental implicit conversion from a `CheckFunction` to a `LightHealth` object.
- **Line 27 — `~LightHealth() override;`**: Declares the destructor. `override` confirms that it overrides a virtual base-class destructor and allows safe destruction through a base pointer.
- **Line 28 — blank separator**: Separates object lifecycle methods from AIDL operations.
- **Line 29 — `triggerHealthCheck() override`**: Declares the generated AIDL method implementation. It schedules a check and returns a Binder transaction status.
- **Line 30 — `registerHealthListener(`**: Begins the declaration of the AIDL listener-registration method.
- **Line 31 — listener parameter and `override`**: Accepts a constant reference to a shared listener interface pointer. `override` verifies that the signature matches the generated AIDL base.
- **Line 32 — `unregisterHealthListener(`**: Begins the listener-unregistration declaration.
- **Line 33 — listener parameter and `override`**: Completes the unregister method declaration with the same shared-interface parameter type.

## Lines 35–45: private helper declarations

- **Line 34 — blank separator**: Separates public Binder operations from private implementation details.
- **Line 35 — `private:`**: Starts members that cannot be called or accessed directly by service clients.
- **Line 36 — `struct ListenerCookie {`**: Declares the small context object passed to the NDK Binder death-recipient API.
- **Line 37 — `LightHealth* service;`**: Stores a non-owning pointer back to the service so the static death callback can reach service state.
- **Line 38 — `AIBinder* binder;`**: Stores the raw Binder identity of the listener that owns this cookie.
- **Line 39 — `};`**: Ends the cookie structure.
- **Line 40 — blank separator**: Separates the cookie type from helper methods.
- **Line 41 — `static void onListenerDied(void* cookie);`**: Declares the Binder death callback. It is static because the C-style NDK API requires a plain callback plus an opaque `void*` context.
- **Line 42 — `void requestHealthCheck();`**: Declares the internal scheduler that sets the pending flag and wakes the worker.
- **Line 43 — `void workerLoop();`**: Declares the function executed by the dedicated worker thread.
- **Line 44 — `LightHealthStatus runHealthCheck();`**: Declares the synchronous operation that validates the core, initializes device I/O, runs the algorithm, and returns an AIDL status.
- **Line 45 — `void notifyListeners(LightHealthStatus status);`**: Declares the method that stores a completed result and invokes registered callbacks.

## Lines 47–52: worker-thread state

- **Line 46 — blank separator**: Separates helper methods from worker data.
- **Line 47 — `std::mutex worker_mutex_;`**: Protects `stopping_`, `check_requested_`, and `check_running_` from concurrent Binder and worker access.
- **Line 48 — `std::condition_variable worker_condition_;`**: Allows the worker to sleep efficiently and be awakened when a request or shutdown arrives.
- **Line 49 — `bool stopping_ = false;`**: Shutdown flag. The destructor sets it to true so the worker exits.
- **Line 50 — `bool check_requested_ = false;`**: Pending-work flag. True means the worker has a health check waiting to start.
- **Line 51 — `bool check_running_ = false;`**: Active-work flag. True means `runHealthCheck()` is currently executing.
- **Line 52 — `std::thread worker_;`**: Owns the dedicated background thread. It is started by the constructor and joined by the destructor.

## Lines 54–62: listener state and class closure

- **Line 53 — blank separator**: Separates worker state from listener state.
- **Line 54 — `std::mutex listener_mutex_;`**: Protects the current status, listener vector, and cookie vector.
- **Line 55 — `current_status_ = UNKNOWN`**: Stores the latest completed AIDL status. Before any check finishes, new listeners receive `UNKNOWN`.
- **Line 56 — `listeners_` vector**: Holds shared ownership of registered listener interfaces, keeping their Binder proxy objects alive.
- **Line 57 — `listener_cookies_` vector**: Owns death-recipient cookies for listeners whose Binder objects successfully linked to death.
- **Line 58 — `death_recipient_`**: RAII wrapper around the NDK Binder death-recipient object that calls `onListenerDied`.
- **Line 59 — `check_function_`**: Stores the optional injected health-check implementation. An empty function selects the real core/device path.
- **Line 60 — `};`**: Ends the `LightHealth` class declaration.
- **Line 61 — blank separator**: Separates the class close from the namespace close.
- **Line 62 — closing namespace**: Ends `aidl::vendor::samsung::microxr::lighthealth`.

### Header state ownership summary

```text
 LightHealth object
 |
 +-- Worker subsystem
 |   +-- worker_mutex_
 |   +-- worker_condition_
 |   +-- stopping_
 |   +-- check_requested_
 |   +-- check_running_
 |   `-- worker_
 |
 +-- Listener subsystem
 |   +-- listener_mutex_
 |   +-- current_status_
 |   +-- listeners_
 |   +-- listener_cookies_
 |   `-- death_recipient_
 |
 `-- Check implementation
     `-- check_function_ (injected) OR hardware/core path
```

---

# Part B — `LightHealth.cpp`

## Lines 1–8: implementation dependencies

- **Line 1 — `#include "LightHealth.h"`**: Imports the class declaration being implemented.
- **Line 2 — `#include "LightHealthDeviceIo.h"`**: Imports the adapter used to control the LED and read ALS values.
- **Line 3 — blank separator**: Separates local project headers from external headers.
- **Line 4 — `#include <android-base/logging.h>`**: Provides Android `LOG(INFO)`, `LOG(WARNING)`, and `LOG(ERROR)` streaming macros.
- **Line 5 — `#include <microxr_light_health_core.h>`**: Imports the C core ABI, version constant, operations structure, result enum, and core run function.
- **Line 6 — blank separator**: Separates platform/core headers from standard C++ headers.
- **Line 7 — `#include <algorithm>`**: Provides `std::find_if` and `std::remove_if` for listener lookup/removal.
- **Line 8 — `#include <utility>`**: Provides `std::move`, used to transfer ownership of the injected function and cookies.

## Lines 10–28: namespace and core-to-AIDL status conversion

- **Line 9 — blank separator**: Ends the include section.
- **Line 10 — implementation namespace**: Opens the same namespace used by the class declaration.
- **Line 11 — blank separator**: Separates the outer namespace from internal helpers.
- **Line 12 — `namespace {`**: Opens an anonymous namespace. Symbols here have translation-unit-only visibility.
- **Line 13 — blank separator**: Separates the anonymous namespace from its helper.
- **Line 14 — `toAidlStatus(...)` signature**: Declares and defines a private file-local function that converts the core C enum into the generated AIDL enum.
- **Line 15 — `switch (status)`**: Selects conversion behavior based on the core result.
- **Line 16 — core `HEALTHY` case**: Matches `MICROXR_LIGHT_HEALTH_HEALTHY`.
- **Line 17 — return AIDL `HEALTHY`**: Converts the healthy core result to the corresponding AIDL value.
- **Line 18 — core `TAMPERED` case**: Matches the core tamper classification.
- **Line 19 — return AIDL `TAMPERED`**: Converts it to the public AIDL value.
- **Line 20 — core `UNKNOWN` case**: Explicitly handles the core unknown value.
- **Line 21 — `default:`**: Also handles any unexpected integer value passed as the enum.
- **Line 22 — return AIDL `UNKNOWN`**: Uses the safest public fallback for both explicit unknown and unexpected values.
- **Line 23 — `}`**: Ends the switch.
- **Line 24 — `}`**: Ends `toAidlStatus`.
- **Line 25 — blank separator**: Separates the helper from the anonymous namespace close.
- **Line 26 — anonymous namespace close**: Keeps `toAidlStatus` private to `LightHealth.cpp`.
- **Line 27 — blank separator**: Separates helpers from class method definitions.
- **Line 28 — `using ndk::ScopedAStatus;`**: Creates a local shorthand for the Binder method result type.

## Lines 30–33: constructor

- **Line 29 — blank separator**: Begins the constructor section.
- **Line 30 — constructor signature**: Defines `LightHealth(CheckFunction)` and receives the optional injected implementation by value.
- **Line 31 — initialize `worker_`**: Constructs and immediately starts the `std::thread`, invoking `LightHealth::workerLoop(this)`.
- **Line 32 — initialize `death_recipient_`**: Creates the native Binder death-recipient object and registers the static `onListenerDied` function as its callback.
- **Line 33 — move `check_function_`; empty body**: Moves the supplied function into the member, avoiding a copy. `{}` means all constructor work is done by member initializers.

> C++ initializes members in the order declared in `LightHealth.h`, not merely the visual order of the initializer list. Here `worker_` is declared before listener state, the death recipient, and `check_function_`.

## Lines 35–44: destructor and worker shutdown

- **Line 34 — blank separator**: Ends the constructor section.
- **Line 35 — destructor signature**: Defines cleanup performed when the Binder service object is destroyed.
- **Line 36 — inner scope open**: Limits the lifetime of the lock guard so the mutex is released before notification and join.
- **Line 37 — lock `worker_mutex_`**: Serializes the shutdown flag update with worker predicate checks.
- **Line 38 — `stopping_ = true`**: Requests termination of `workerLoop()`.
- **Line 39 — inner scope close**: Destroys the lock guard and releases `worker_mutex_`.
- **Line 40 — notify worker**: Wakes the worker if it is sleeping on the condition variable.
- **Line 41 — `worker_.joinable()` check**: Verifies that the thread represents a running or completed thread that has not yet been joined.
- **Line 42 — `worker_.join()`**: Waits for `workerLoop()` to return before destruction proceeds, preventing the thread from using a destroyed object.
- **Line 43 — `}`**: Ends the joinability condition.
- **Line 44 — `}`**: Ends the destructor.

## Lines 46–49: Binder trigger entry point

- **Line 45 — blank separator**: Begins the trigger method.
- **Line 46 — method signature**: Defines the AIDL Binder method called by a client to request a check.
- **Line 47 — `requestHealthCheck()`**: Delegates scheduling to the internal flag/condition-variable method.
- **Line 48 — return `ScopedAStatus::ok()`**: Reports successful acceptance of the trigger. This does not contain the health result; the result is delivered asynchronously to listeners.
- **Line 49 — `}`**: Ends the trigger method.

## Lines 51–61: request scheduling and coalescing

- **Line 50 — blank separator**: Begins the private scheduling helper.
- **Line 51 — function signature**: Defines `requestHealthCheck()`.
- **Line 52 — inner scope open**: Limits the worker lock lifetime.
- **Line 53 — lock `worker_mutex_`**: Protects the request/running flags while they are inspected and changed.
- **Line 54 — pending-or-running test**: Detects whether a check is already queued or currently executing.
- **Line 55 — informational log**: Records that this additional trigger will be merged into the existing operation.
- **Line 56 — `return`**: Leaves without creating another pending job.
- **Line 57 — `}`**: Ends the coalescing condition.
- **Line 58 — `check_requested_ = true`**: Marks one check as pending when the service was idle.
- **Line 59 — inner scope close**: Releases `worker_mutex_` before waking the worker.
- **Line 60 — `notify_one()`**: Wakes one waiter; this service has one dedicated worker waiter.
- **Line 61 — `}`**: Ends `requestHealthCheck()`.

## Lines 63–82: worker loop

- **Line 62 — blank separator**: Begins worker execution logic.
- **Line 63 — function signature**: Defines the function used as the `std::thread` entry point.
- **Line 64 — `while (true)`**: Keeps the worker alive for the service lifetime.
- **Line 65 — inner scope open**: Contains the lock used for waiting and state transition.
- **Line 66 — `std::unique_lock`**: Locks `worker_mutex_`. A unique lock is required because `condition_variable::wait` temporarily unlocks and later relocks it.
- **Line 67 — condition-variable wait**: Sleeps until notified and the predicate says either shutdown or pending work exists. The predicate also makes spurious wakeups harmless.
- **Line 68 — shutdown test**: Checks `stopping_` while the mutex is held.
- **Line 69 — `return`**: Exits the worker thread. Shutdown takes precedence over pending work when both flags are true.
- **Line 70 — `}`**: Ends the shutdown condition.
- **Line 71 — clear pending flag**: Consumes the queued request before starting it.
- **Line 72 — set running flag**: Marks the long-running check as active, causing new triggers to be coalesced.
- **Line 73 — inner scope close**: Releases the worker mutex before hardware work, so Binder threads can inspect scheduling state.
- **Line 74 — blank separator**: Separates state transition from the actual check.
- **Line 75 — run synchronous check**: Executes `runHealthCheck()` on the worker and stores the returned AIDL status.
- **Line 76 — inner scope open**: Begins the protected running-state update.
- **Line 77 — lock worker state**: Prevents races with concurrent trigger scheduling.
- **Line 78 — clear running flag**: Marks the check complete.
- **Line 79 — inner scope close**: Releases the worker lock.
- **Line 80 — notify listeners**: Stores and sends the result after `check_running_` becomes false.
- **Line 81 — `}`**: Returns to the top of the infinite loop and waits for the next request.
- **Line 82 — `}`**: Ends `workerLoop()`.

## Lines 84–115: health-check execution

- **Line 83 — blank separator**: Begins the check implementation.
- **Line 84 — function signature**: Defines the synchronous method returning the public AIDL status type.
- **Line 85 — injected-function test**: `std::function` converts to true when it contains a callable target.
- **Line 86 — invoke injected function**: Returns the injected result immediately. This bypasses API-version checks, device I/O, LED operations, and the real core.
- **Line 87 — `}`**: Ends the test path.
- **Line 88 — blank separator**: Separates test-injection behavior from production behavior.
- **Line 89 — read core API version**: Calls the shared core library to discover the runtime ABI version.
- **Line 90 — compare versions**: Requires the runtime version to equal the header's compile-time `MICROXR_LIGHT_HEALTH_CORE_API_VERSION`.
- **Line 91 — version error log**: Records the unsupported version number.
- **Line 92 — return `UNKNOWN`**: Stops because the service cannot safely use a mismatched core ABI.
- **Line 93 — `}`**: Ends the version mismatch branch.
- **Line 94 — blank separator**: Separates version validation from device setup.
- **Line 95 — construct `device_io`**: Creates a per-check adapter on the stack. Its destructor later performs sensor/LED cleanup.
- **Line 96 — initialize device I/O**: Connects to Light Behaviors and initializes/selects the ALS path.
- **Line 97 — initialization failure test**: Android-style success is zero; any nonzero value is treated as failure.
- **Line 98 — begin initialization error log**: Writes the descriptive prefix.
- **Line 99 — append error code**: Adds the numeric initialization result to the same log message.
- **Line 100 — return `UNKNOWN`**: No algorithm is run when its hardware callbacks cannot be initialized.
- **Line 101 — `}`**: Ends the initialization-error branch.
- **Line 102 — blank separator**: Separates initialization from core execution.
- **Line 103 — build `MicroXrLightHealthOps`**: Obtains the C callback table whose context points to this `device_io` instance.
- **Line 104 — run core algorithm**: Passes the operations table to the core and stores its C enum result.
- **Line 105 — blank separator**: Separates core execution from final LED cleanup.
- **Line 106 — comment, first half**: Documents service ownership of hardware callbacks.
- **Line 107 — comment, second half**: States that the service attempts to leave the LED off even if the core returned an error status.
- **Line 108 — final LED-OFF request**: Calls the device adapter directly rather than relying only on the core.
- **Line 109 — OFF failure test**: Checks whether the Binder/device request succeeded.
- **Line 110 — OFF failure log**: Records the final LED cleanup error code.
- **Line 111 — return `UNKNOWN`**: Overrides the core classification if the final OFF request fails.
- **Line 112 — `}`**: Ends the LED-off failure branch.
- **Line 113 — blank separator**: Separates cleanup from return conversion.
- **Line 114 — convert and return result**: Maps the core C enum to the AIDL enum using `toAidlStatus()`.
- **Line 115 — `}`**: Ends `runHealthCheck()`. The stack `device_io` is destroyed here.

## Lines 117–149: listener registration

- **Line 116 — blank separator**: Begins listener registration.
- **Line 117 — method signature, first line**: Starts the AIDL registration method definition.
- **Line 118 — listener parameter**: Receives a shared interface pointer by constant reference.
- **Line 119 — null test**: Validates that the client supplied a listener object.
- **Line 120 — illegal-argument status**: Returns an AIDL/Binder exception instead of dereferencing null.
- **Line 121 — `}`**: Ends null validation.
- **Line 122 — blank separator**: Separates validation from Binder identity extraction.
- **Line 123 — obtain raw Binder identity**: Converts the listener interface to its underlying Binder and stores the raw `AIBinder*` for identity comparisons/death linkage.
- **Line 124 — local `current_status` declaration**: Reserves a value to copy while locked and deliver after unlocking.
- **Line 125 — inner scope open**: Contains all listener-collection operations requiring `listener_mutex_`.
- **Line 126 — lock listener state**: Serializes registration with notification, unregistration, and death cleanup.
- **Line 127 — begin `std::find_if`**: Starts searching existing listener entries.
- **Line 128 — iterator range**: Searches from the beginning to the end of `listeners_`.
- **Line 129 — Binder-identity predicate**: Captures the candidate Binder pointer and compares it with each stored listener's Binder pointer.
- **Line 130 — duplicate test**: Determines whether a listener with the same Binder identity is already registered.
- **Line 131 — return OK for duplicate**: Makes registration idempotent and does not add a second vector entry.
- **Line 132 — `}`**: Ends the duplicate branch.
- **Line 133 — blank separator**: Separates duplicate handling from death-link setup.
- **Line 134 — allocate `ListenerCookie`**: Creates a uniquely owned cookie containing the service pointer and listener Binder pointer.
- **Line 135 — declare link status**: Begins storing the result from the NDK death-link call.
- **Line 136 — `AIBinder_linkToDeath(...)`**: Associates this Binder with `death_recipient_` and passes the cookie pointer back to the future death callback.
- **Line 137 — accepted-status test**: Accepts normal success and `STATUS_INVALID_OPERATION`. The latter commonly means death linkage is not applicable, such as for a local Binder.
- **Line 138 — return other Binder error**: Converts any other native Binder status into a `ScopedAStatus`.
- **Line 139 — `}`**: Ends death-link error handling.
- **Line 140 — store listener**: Adds shared ownership of the listener to the registered-listener vector.
- **Line 141 — successful death-link test**: Only a successfully linked Binder needs a persistent cookie.
- **Line 142 — move cookie into vector**: Transfers unique ownership into `listener_cookies_`, ensuring the opaque pointer remains valid while linked.
- **Line 143 — `}`**: Ends cookie storage condition.
- **Line 144 — copy latest status**: Reads `current_status_` while listener state is locked.
- **Line 145 — inner scope close**: Unlocks before making a Binder callback to the listener.
- **Line 146 — blank separator**: Separates protected state update from external callback.
- **Line 147 — immediate status callback**: Sends the newly registered listener the current known status, initially `UNKNOWN` if no check has completed.
- **Line 148 — return OK**: Reports successful registration after making the initial callback.
- **Line 149 — `}`**: Ends `registerHealthListener()`.

## Lines 151–173: listener unregistration

- **Line 150 — blank separator**: Begins listener unregistration.
- **Line 151 — method signature, first line**: Starts the AIDL unregister method definition.
- **Line 152 — listener parameter**: Receives the listener interface to remove.
- **Line 153 — null test**: Validates the argument.
- **Line 154 — illegal-argument result**: Returns the matching Binder exception for null.
- **Line 155 — `}`**: Ends validation.
- **Line 156 — blank separator**: Separates validation from removal.
- **Line 157 — obtain Binder identity**: Uses the same raw Binder pointer identity used during registration.
- **Line 158 — lock listener state**: Keeps listener and cookie lookup/removal atomic relative to notification and death cleanup.
- **Line 159 — begin listener lookup**: Starts searching the listener vector.
- **Line 160 — listener iterator range**: Covers all registered listeners.
- **Line 161 — listener identity predicate**: Finds the listener whose underlying Binder pointer matches the argument.
- **Line 162 — begin cookie lookup**: Starts an independent search of the cookie vector.
- **Line 163 — cookie iterator range**: Covers all stored death-link cookies.
- **Line 164 — cookie identity predicate**: Finds the cookie whose stored Binder pointer matches.
- **Line 165 — cookie-found test**: Unlinking is only needed if a successfully linked cookie exists.
- **Line 166 — unlink death recipient**: Removes the Binder death association using the exact recipient and cookie pointer used during linkage.
- **Line 167 — erase cookie**: Destroys the cookie after unlinking.
- **Line 168 — `}`**: Ends cookie removal.
- **Line 169 — listener-found test**: Allows unregistration of a nonregistered listener to remain harmless.
- **Line 170 — erase listener**: Removes the shared listener interface from the vector.
- **Line 171 — `}`**: Ends listener removal.
- **Line 172 — return OK**: Reports successful idempotent unregistration.
- **Line 173 — `}`**: Ends `unregisterHealthListener()`.

## Lines 175–189: completed-result notification

- **Line 174 — blank separator**: Begins listener notification.
- **Line 175 — method signature**: Defines notification using a completed AIDL status.
- **Line 176 — local listener-vector copy**: Creates a temporary vector that will keep callbacks alive after the mutex is released.
- **Line 177 — inner scope open**: Contains protected status/list operations.
- **Line 178 — lock listener state**: Synchronizes with registration, unregistration, and Binder death.
- **Line 179 — update `current_status_`**: Makes this result the status immediately delivered to future registrations.
- **Line 180 — copy `listeners_`**: Takes shared references to the current listeners while locked.
- **Line 181 — inner scope close**: Releases `listener_mutex_` before external Binder callbacks.
- **Line 182 — iterate copied listeners**: Visits every listener present in the snapshot.
- **Line 183 — call `onLightHealth(status)`**: Delivers the result and stores the listener transaction's `ScopedAStatus`.
- **Line 184 — callback failure test**: Checks whether the remote transaction or listener method failed.
- **Line 185 — warning log prefix**: Begins a nonfatal callback-failure message.
- **Line 186 — append description**: Adds Binder's human-readable error description.
- **Line 187 — `}`**: Ends callback-failure handling.
- **Line 188 — `}`**: Ends the listener loop.
- **Line 189 — `}`**: Ends `notifyListeners()`.

## Lines 191–214: Binder death callback

- **Line 190 — blank separator**: Begins automatic dead-listener removal.
- **Line 191 — static callback signature**: Defines the C-compatible death callback registered in the constructor.
- **Line 192 — null-cookie test**: Protects against invalid callback context.
- **Line 193 — `return`**: Does nothing when no context is available.
- **Line 194 — `}`**: Ends null validation.
- **Line 195 — cast opaque cookie**: Converts the `void*` supplied by Binder back into `ListenerCookie*`.
- **Line 196 — recover service pointer**: Retrieves the `LightHealth` instance that owns listener state.
- **Line 197 — recover listener Binder pointer**: Retrieves the identity to remove.
- **Line 198 — blank separator**: Separates context extraction from synchronized removal.
- **Line 199 — lock service listener state**: Uses the recovered service pointer to lock its private mutex.
- **Line 200 — begin listener erase**: Uses the erase-remove idiom to delete matching listener entries.
- **Line 201 — `std::remove_if` range**: Scans the complete listener vector and moves nonmatching entries forward.
- **Line 202 — predicate declaration**: Captures the dead Binder pointer and receives each listener.
- **Line 203 — compare Binder identities**: Marks listeners backed by the dead Binder for removal.
- **Line 204 — `}`**: Ends the predicate body.
- **Line 205 — erase tail through end**: Erases all elements placed in the removable tail by `remove_if`.
- **Line 206 — begin cookie erase**: Applies the same erase-remove pattern to owned death cookies.
- **Line 207 — cookie `remove_if` range**: Scans all stored cookies.
- **Line 208 — cookie predicate declaration**: Captures the dead Binder and receives each `unique_ptr<ListenerCookie>`.
- **Line 209 — compare stored Binder**: Marks cookies associated with that dead Binder.
- **Line 210 — `}`**: Ends the cookie predicate body.
- **Line 211 — erase cookie tail**: Destroys every matching cookie and keeps unrelated cookies.
- **Line 212 — `}`**: Ends `onListenerDied()`; the listener mutex unlocks as the lock guard is destroyed.
- **Line 213 — blank separator**: Separates the last method from namespace closure.
- **Line 214 — closing namespace**: Ends the implementation namespace and the source file.

---

# 3. Combined runtime flow

```text
 Client Binder thread
        |
        | triggerHealthCheck()
        v
 requestHealthCheck()
        |
        | worker_mutex_
        | check_requested_=true
        v
 worker_condition_.notify_one()
        |
        v
 workerLoop()
        |
        | check_requested_=false
        | check_running_=true
        v
 runHealthCheck()
        |
        +--> injected CheckFunction? -- yes --> return injected status
        |
        `--> production path
             |
             +--> verify core API version
             +--> initialize LightHealthDeviceIo
             +--> obtain MicroXrLightHealthOps
             +--> run core algorithm
             +--> force final LED OFF
             `--> convert core status to AIDL status
        |
        v
 check_running_=false
        |
        v
 notifyListeners(status)
        |
        +--> current_status_=status
        +--> copy listener vector while locked
        `--> call onLightHealth(status) without listener lock
```

# 4. Why there are two mutexes

```text
 worker_mutex_                              listener_mutex_
 +----------------------------------+       +----------------------------------+
 | stopping_                        |       | current_status_                  |
 | check_requested_                 |       | listeners_                       |
 | check_running_                   |       | listener_cookies_                |
 +----------------------------------+       +----------------------------------+
 | Used by Binder trigger + worker  |       | Used by Binder register/remove,  |
 | condition-variable transitions  |       | notification, and death callback |
 +----------------------------------+       +----------------------------------+
```

Keeping these state domains separate means a listener registration does not need to acquire the worker scheduling lock, and scheduling a health check does not need to acquire the listener collection lock.

---

# 5. Where the AIDL-generated classes come from

The current workspace contains the HAL implementation but does not contain the original `.aidl` source files or the `aidl_interface` module. In a complete Android source tree, those normally exist in a separate interface directory similar to:

```text
<interface-module>/
├── Android.bp
├── aidl/
│   └── vendor/google/microxr/lighthealth/
│       ├── ILightHealth.aidl
│       ├── ILightHealthListener.aidl
│       └── LightHealthStatus.aidl
└── aidl_api/
    └── vendor.google.microxr.lighthealth/
        ├── 1/
        └── current/
```

The package in `ILightHealth.aidl` is expected to be:

```aidl
package vendor.google.microxr.lighthealth;
```

An `aidl_interface` Soong module generates the versioned NDK library used by this HAL:

```text
vendor.google.microxr.lighthealth-V1-ndk
```

The module naming parts mean:

```text
vendor.google.microxr.lighthealth  = AIDL interface module name
V1                                 = frozen interface version 1
ndk                                = native C/C++ AIDL backend
```

The NDK backend generates files conceptually like:

```text
ILightHealth.aidl
   |
   +--> ILightHealth.h       common interface and descriptor
   +--> BnLightHealth.h      native server stub
   +--> BpLightHealth.h      native client proxy
   `--> generated .cpp       Binder parcel and transaction code

ILightHealthListener.aidl
   |
   +--> ILightHealthListener.h
   +--> BnLightHealthListener.h
   +--> BpLightHealthListener.h
   `--> generated .cpp

LightHealthStatus.aidl
   |
   `--> LightHealthStatus.h and generated support code
```

## Build-output location

Generated headers are normally placed under Soong intermediates rather than beside `LightHealth.h`:

```text
out/soong/.intermediates/<interface-source-location>/
    vendor.google.microxr.lighthealth-V1-ndk-source/
        gen/include/
            aidl/vendor/google/microxr/lighthealth/
                ILightHealth.h
                BnLightHealth.h
                BpLightHealth.h
                ILightHealthListener.h
                BnLightHealthListener.h
                BpLightHealthListener.h
                LightHealthStatus.h
```

The exact prefix depends on the location of the interface module and the Android release. It can be found after a build with:

```bash
find out/soong/.intermediates -name BnLightHealth.h
```

Generated headers are build-time inputs. They are not expected to be installed as headers on the Android device.

---

# 6. How the generated include path is resolved

`LightHealth.h` contains:

```cpp
#include <aidl/vendor/google/microxr/lighthealth/BnLightHealth.h>
```

This is a path relative to a compiler include root. The HAL's `Android.bp` declares:

```bp
shared_libs: [
    "vendor.google.microxr.lighthealth-V1-ndk",
]
```

That dependency exports the generated include directory to the HAL build. Conceptually, Soong invokes the compiler with an option like:

```text
-I out/soong/.../gen/include
```

The compiler resolves the header by combining the exported root and the include text:

```text
out/soong/.../gen/include
+
aidl/vendor/google/microxr/lighthealth/BnLightHealth.h
=
out/soong/.../gen/include/aidl/vendor/google/microxr/lighthealth/BnLightHealth.h
```

The path is known from the AIDL package and generated class name:

```text
AIDL package:
vendor.google.microxr.lighthealth

Dots converted to directories:
vendor/google/microxr/lighthealth

NDK generated include prefix:
aidl/vendor/google/microxr/lighthealth

Server class generated from ILightHealth:
BnLightHealth.h
```

Therefore, the complete logical include is:

```text
aidl/vendor/google/microxr/lighthealth/BnLightHealth.h
```

---

# 7. Build-time headers versus the runtime `.so`

The generated header and generated shared library serve different stages:

```text
BnLightHealth.h
    = used by the compiler while building the HAL

vendor.google.microxr.lighthealth-V1-ndk.so
    = used by the dynamic linker when the HAL runs
```

Because the HAL lists the AIDL module under `shared_libs`, Soong builds/links a shared-library variant named approximately:

```text
vendor.google.microxr.lighthealth-V1-ndk.so
```

Because the service is a vendor module:

```bp
vendor: true,
```

a 64-bit vendor variant is normally available under:

```text
/vendor/lib64/vendor.google.microxr.lighthealth-V1-ndk.so
```

A 32-bit vendor variant would normally be under:

```text
/vendor/lib/vendor.google.microxr.lighthealth-V1-ndk.so
```

The installation path is selected by Soong; it is not hardcoded in `LightHealth.cpp`. Actual device output can be checked with:

```bash
adb shell find /vendor/lib64 -name '*lighthealth*'
```

The service's runtime dependencies can be inspected with:

```bash
adb shell ldd \
    /vendor/bin/hw/vendor.samsung.microxr.lighthealth-service.default
```

Complete build/runtime relationship:

```text
AIDL source files
       |
       | NDK AIDL generation
       +-----------------------------+
       |                             |
       v                             v
Generated headers              Generated Binder code
       |                             |
       | compile                     | compile/link
       +--------------+--------------+
                      v
        HAL executable + versioned NDK .so
                      |
                      v
              Android runtime
```

---

# 8. `Bn` server stub and `Bp` client proxy

For an interface named `ILightHealth`, the NDK backend uses these naming conventions:

```text
ILightHealth    = common interface type
BnLightHealth  = Binder-native server stub
BpLightHealth  = Binder client proxy
```

The primary call direction is:

```text
CLIENT PROCESS                           HAL PROCESS

+----------------------+                 +----------------------+
| BpLightHealth        |                 | BnLightHealth        |
| generated proxy      |                 | generated stub       |
+----------+-----------+                 +----------+-----------+
           |                                        |
           | Binder transaction                     | dispatch
           +------------> Binder driver ------------+
                                                    |
                                                    v
                                         +----------------------+
                                         | LightHealth          |
                                         | implementation       |
                                         +----------------------+
```

`LightHealth` is the real implementation because it inherits the server stub:

```cpp
class LightHealth final
    : public ::aidl::vendor::google::microxr::lighthealth::BnLightHealth {
```

The generated `BnLightHealth` handles transaction decoding and calls the overrides in `LightHealth`. The implementation does not need to manually decode parcels or implement `onTransact()`.

Listener callbacks travel in the reverse direction:

```text
HAL PROCESS                              CLIENT PROCESS

ILightHealthListener
usually backed by                       BnLightHealthListener
BpLightHealthListener                   implemented by client
        |                                       ^
        | onLightHealth(status)                 |
        +------------> Binder driver -----------+
```

This is why the HAL can call:

```cpp
listener->onLightHealth(status);
```

as if it were a normal C++ method even when the listener lives in another process.

---

# 9. Why the implementation namespace uses `samsung`

The header declares:

```cpp
namespace aidl::vendor::samsung::microxr::lighthealth {
```

A C++ namespace is a name-organizing scope. It does not need to exist before this line. The declaration creates or reopens the nested namespaces and is equivalent to:

```cpp
namespace aidl {
namespace vendor {
namespace samsung {
namespace microxr {
namespace lighthealth {

// declarations

}
}
}
}
}
```

A C++ namespace does not require a matching filesystem directory and does not create a runtime object.

This project uses two different namespaces:

```text
Generated AIDL namespace:
aidl::vendor::google::microxr::lighthealth

Implementation namespace:
aidl::vendor::samsung::microxr::lighthealth
```

They are connected by inheritance:

```text
::aidl::vendor::google::microxr::lighthealth::BnLightHealth
                              ^
                              |
                              | inherited by
                              |
aidl::vendor::samsung::microxr::lighthealth::LightHealth
```

The leading `::` in the generated base name means "start from the global C++ namespace." It prevents relative namespace lookup from changing the meaning.

The Samsung namespace is manually selected implementation organization. It is not generated from the Google AIDL package. It could be changed to another implementation namespace if all implementation references were changed consistently.

`LightHealth.cpp` opens the same namespace so its short method definitions refer to the class declared in `LightHealth.h`:

```cpp
namespace aidl::vendor::samsung::microxr::lighthealth {

LightHealth::LightHealth(...) {
    ...
}

}
```

`service.cpp` imports the implementation class using its complete name:

```cpp
using ::aidl::vendor::samsung::microxr::lighthealth::LightHealth;
```

The C++ implementation namespace does not determine the Binder descriptor, service-manager name, VINTF identity, executable name, or filesystem installation path.

---

# 10. Binder descriptor

A Binder descriptor is a unique string that identifies the interface type. It is derived from:

```text
AIDL package + "." + AIDL interface name
```

For this HAL:

```text
AIDL package:
vendor.google.microxr.lighthealth

Interface:
ILightHealth

Descriptor:
vendor.google.microxr.lighthealth.ILightHealth
```

The AIDL compiler generates a member conceptually available as:

```cpp
ILightHealth::descriptor
```

Because `LightHealth` inherits through `BnLightHealth`, the descriptor is also accessible as:

```cpp
LightHealth::descriptor
```

The inheritance/access path is:

```text
ILightHealth
    ^
    |
BnLightHealth
    ^
    |
LightHealth

LightHealth::descriptor
    -> vendor.google.microxr.lighthealth.ILightHealth
```

The descriptor identifies what interface a Binder object implements. It assists generated proxy/stub code, interface association, and service lookup. It is not a filename, library name, process name, or C++ namespace.

---

# 11. Meaning of `/default`

AIDL Binder service names use this structure:

```text
<interface descriptor>/<instance name>
```

For this project:

```text
Descriptor:
vendor.google.microxr.lighthealth.ILightHealth

Instance name:
default

Complete registered service name:
vendor.google.microxr.lighthealth.ILightHealth/default
```

The `/default` text is not a filesystem path. It identifies one instance of the interface. Android permits multiple implementations of the same interface, for example:

```text
vendor.google.microxr.lighthealth.ILightHealth/default
vendor.google.microxr.lighthealth.ILightHealth/secondary
vendor.google.microxr.lighthealth.ILightHealth/test
```

`default` is the conventional name when there is one normal implementation.

The instance must match across the service and Android configuration:

```text
service.cpp
  LightHealth::descriptor + "/default"
                      |
                      +-----------------------------------+
                      |                                   |
                      v                                   v
init RC                                           VINTF manifest
...ILightHealth/default                          <instance>default</instance>
                      |
                      v
client lookup
ILightHealth::descriptor + "/default"
```

A different instance name can be used, but registration, client lookup, RC declaration, and VINTF declaration must all use the same name.

---

# 12. Binder service versus HAL

`LightHealth` is both a HAL and a Binder service, but those terms describe different aspects:

```text
HAL            = the role: abstract hardware-specific behavior
Binder service = the IPC mechanism: expose callable methods across processes
```

The HAL role is to hide details of the LED, ALS, Light Behaviors service, sensor selection, and core algorithm behind `ILightHealth`.

The Binder service role is to publish an object to Service Manager and receive cross-process method calls.

```text
Android client/framework
          |
          | AIDL Binder call
          v
+-------------------------------------------------------+
| Light Health AIDL HAL                                 |
|                                                       |
| Binder-facing layer                                   |
|   service.cpp + LightHealth.h/.cpp                    |
|                                                       |
| Hardware-facing layer                                 |
|   LightHealthDeviceIo.h/.cpp + core/                  |
+----------------------+--------------------------------+
                       |
             +---------+---------+
             |                   |
             v                   v
     Light Behaviors           Sensor Manager
             |                   |
             v                   v
            LED                 ALS
```

Not every Binder service is a HAL; many Binder services implement software-only functionality. This component is a HAL because it is a versioned device interface, is declared in VINTF, is built for the vendor partition, and abstracts device behavior. It is a Binder service because it uses generated `Bn`/`Bp` code and registers with Service Manager.

## Init service is a third concept

The RC file defines an init-managed process:

```text
Init service name:
vendor.lighthealth-default

Executable:
/vendor/bin/hw/vendor.samsung.microxr.lighthealth-service.default

Binder service published by that process:
vendor.google.microxr.lighthealth.ILightHealth/default
```

These names refer to different layers:

| Layer | Name | Function |
|---|---|---|
| Android init service | `vendor.lighthealth-default` | Starts and supervises the Linux process. |
| Executable/process | `vendor.samsung.microxr.lighthealth-service.default` | Contains and executes the HAL code. |
| Binder service | `vendor.google.microxr.lighthealth.ILightHealth/default` | Allows clients to find and call the HAL. |
| C++ class | `aidl::vendor::samsung::microxr::lighthealth::LightHealth` | Implements the AIDL operations. |

---

# 13. Where the Binder service resides at runtime

The real Binder service object resides in the HAL process memory.

This line allocates it in the HAL process:

```cpp
auto service = ndk::SharedRefBase::make<LightHealth>();
```

Conceptually:

```text
HAL PROCESS HEAP
+-------------------------------------------------------+
| LightHealth object                                    |
|                                                       |
| - generated BnLightHealth base/stub                   |
| - worker thread state                                 |
| - current status                                      |
| - listener vector                                     |
| - Binder death-recipient state                        |
+-------------------------------------------------------+
```

The next call registers a Binder reference to that object:

```cpp
AServiceManager_addService(
    service->asBinder().get(),
    instance.c_str());
```

Service Manager stores a mapping from the name to a Binder reference. It does not copy the implementation object into the Service Manager process.

```text
SERVICE MANAGER PROCESS
+-------------------------------------------------------+
| Name                                                  |
| vendor.google.microxr.lighthealth.ILightHealth/default|
|                                                       |
|       maps to                                         |
|          |                                            |
|          v                                            |
| Binder reference to object in HAL process             |
+-------------------------------------------------------+
```

The kernel Binder driver tracks Binder nodes and per-process handles and routes transactions. It does not execute the `LightHealth` C++ implementation itself.

A client process contains a generated proxy, not the real object:

```text
+----------------------+     +------------------+     +----------------------+
| CLIENT PROCESS       |     | KERNEL           |     | HAL PROCESS          |
|                      |     | Binder driver    |     |                      |
| BpLightHealth proxy  |---->| transaction      |---->| BnLightHealth stub   |
|                      |     | routing          |     |       |              |
+----------------------+     +------------------+     |       v              |
                                                       | LightHealth methods  |
                                                       +----------------------+
```

`ABinderProcess_joinThreadPool()` keeps the HAL process available to receive these transactions. A Binder-pool thread in that same HAL process receives a transaction, the generated `BnLightHealth` dispatches it, and the actual `LightHealth` override runs there.

## Complete process-level call sequence

```text
1. Android init starts the HAL executable.

2. The HAL process constructs LightHealth.

3. The HAL process registers its Binder object under:
   vendor.google.microxr.lighthealth.ILightHealth/default

4. Service Manager stores name -> Binder reference.

5. A client asks Service Manager for the same name.

6. The client receives a Binder handle and ILightHealth::fromBinder()
   creates/returns an interface backed by BpLightHealth.

7. The client calls triggerHealthCheck().

8. BpLightHealth encodes a Binder transaction.

9. The kernel Binder driver routes it to a Binder-pool thread in the
   HAL process.

10. BnLightHealth decodes the transaction and calls
    LightHealth::triggerHealthCheck().

11. LightHealth schedules the long operation on its worker thread,
    which is also inside the HAL process.
```

The final ownership summary is:

```text
Real LightHealth implementation  -> HAL process
Generated BnLightHealth stub     -> HAL process
Health-check worker thread       -> HAL process
Service name/reference mapping   -> Service Manager process
Transaction routing              -> kernel Binder driver
Generated BpLightHealth proxy    -> client process
```

---

# 14. Why this HAL uses many NDK-related dependencies

The service is a native C++ executable rather than a Java service. It performs three native operations:

```text
1. Publishes a native AIDL Binder service
2. Calls another native AIDL Binder service
3. Reads an Android sensor from native code
```

Those operations explain the NDK-related dependencies in `Android.bp`:

```text
vendor.samsung.microxr.lighthealth-service.default
|
+-- libbinder_ndk
|   `-- Service Manager, Binder pool, Binder objects, statuses,
|       death notifications, and native Binder ownership utilities
|
+-- vendor.google.microxr.lighthealth-V1-ndk
|   `-- Generated ILightHealth server stub, listener interface,
|       proxy code, enum, and parcel/transaction code
|
+-- vendor.google.microxr.lightbehaviors-V1-ndk
|   `-- Generated client proxy and data types used to control the LED
|
+-- libsensorndkbridge
|   `-- ASensorManager, ASensorEventQueue, ASensorEvent, and looper path
|
+-- libmicroxr_light_health_core
|   `-- Device-independent light-health algorithm
|
+-- libbase
|   `-- Android LOG and CHECK helpers
|
`-- liblog
    `-- Native Android logging implementation
```

## Which AIDL backend is used?

Both generated AIDL dependencies explicitly select the NDK backend:

```bp
"vendor.google.microxr.lighthealth-V1-ndk",
"vendor.google.microxr.lightbehaviors-V1-ndk",
```

The `-ndk` suffix is the direct indication. Source-code evidence includes:

```cpp
ndk::ScopedAStatus
ndk::SharedRefBase
ndk::SpAIBinder
ndk::ScopedAIBinder_DeathRecipient
AIBinder*
```

and the Binder runtime dependency is:

```bp
"libbinder_ndk",
```

A representative interface configuration is:

```bp
aidl_interface {
    name: "vendor.google.microxr.lighthealth",
    stability: "vintf",

    backend: {
        ndk: {
            enabled: true,
        },
        cpp: {
            enabled: false,
        },
        java: {
            enabled: false,
        },
    },
}
```

The actual interface module is outside the current workspace, so its exact properties must be checked in the complete Android tree.

## NDK backend still generates C++

The word `ndk` does not mean that the service is written only in C. Both native backends generate C++ interfaces; they use different Binder APIs:

| AIDL backend | Typical generated types | Runtime library | Intended environment |
|---|---|---|---|
| NDK — used here | `ndk::ScopedAStatus`, `AIBinder*`, `std::shared_ptr` | `libbinder_ndk` | Stable native/vendor boundary |
| CPP platform backend | `android::binder::Status`, `android::sp`, `android::IBinder` | `libbinder` | Platform-internal native services |
| Java backend | Java `Stub`, `Proxy`, and interface classes | Java Binder runtime | Java services/clients |
| Rust backend | Rust Binder interfaces and types | Rust Binder libraries | Native Rust services/clients |

```text
ILightHealth.aidl
       |
       +-- NDK backend --> C++ using libbinder_ndk  <-- selected here
       |
       `-- CPP backend --> C++ using platform libbinder
```

## Why not use only the CPP backend?

The module is a vendor HAL:

```bp
vendor: true,
```

A vendor HAL may remain installed while the system partition is updated. The NDK Binder layer is designed to provide the stable native ABI needed at this system/vendor boundary.

```text
Before system OTA:

System image version A  <---- stable AIDL/NDK ---->  Vendor HAL

After system OTA:

System image version B  <---- same stable contract -> Same vendor HAL
```

The platform CPP backend uses Android's internal `libbinder` C++ API and is more closely tied to platform implementation and C++ ABI details. It is suitable when components are platform-internal and updated together. The NDK backend is the appropriate choice for a VINTF-stable native vendor HAL.

Two separate stability layers work together:

```text
Stable AIDL V1
    protects method signatures, enum values, parcel layout,
    and Binder transaction compatibility

Stable libbinder_ndk ABI
    protects the native functions/types used by the compiled vendor binary
```

An interface can enable several backends for different consumers, but this service specifically links the generated `-V1-ndk` variant.

---

# 15. Which APIs in `service.cpp` are stable?

Most Binder-facing operations in `service.cpp` use the stable native Binder API, but not every included header is a stable NDK API.

| Header/symbol | Category | Stability meaning |
|---|---|---|
| `<android/binder_manager.h>` | `libbinder_ndk` Service Manager API | Stable native Binder API, subject to Android API availability |
| `<android/binder_process.h>` | `libbinder_ndk` process/pool API | Stable native Binder API, subject to Android API availability |
| `AServiceManager_addService()` | Native service registration | `libbinder_ndk` API |
| `ABinderProcess_*()` | Native Binder pool management | `libbinder_ndk` API |
| `AIBinder*`, `STATUS_OK` | Native Binder types/status | `libbinder_ndk` API |
| `ndk::SharedRefBase` | Native Binder ownership utility | `libbinder_ndk` C++ utility |
| Generated `BnLightHealth` | Versioned generated interface | Stable when backed by frozen/VINTF-stable AIDL |
| `<android-base/logging.h>` | Android `libbase` utility | Platform/vendor utility, not a public NDK stability contract |
| `CHECK_EQ` | `libbase` fatal-check macro | Implementation utility, not a Binder API |
| `<cstdlib>`, `<string>` | Standard C/C++ library | Standard language/library APIs |
| `LightHealth.h` | Local implementation | Private implementation API, not client ABI |

The important separation is:

```text
Client-visible stable contract
    ILightHealth AIDL V1 + generated NDK transaction code

Native Binder runtime contract
    libbinder_ndk

Private implementation
    LightHealth, worker logic, listener containers, logging choices
```

Clients should depend on the AIDL interface, not the private `LightHealth` C++ class.

---

# 16. `service.cpp` startup explained line by line

The file creates, publishes, and keeps the real HAL service alive.

## Includes

```cpp
#include <android-base/logging.h>
```

Provides `CHECK_EQ`, which terminates the process with a fatal log if service registration fails.

```cpp
#include <android/binder_manager.h>
```

Provides `AServiceManager_addService()` and other NDK Service Manager operations.

```cpp
#include <android/binder_process.h>
```

Provides NDK Binder thread-pool configuration and join functions.

```cpp
#include <cstdlib>
#include <string>
```

Provide `EXIT_FAILURE` and `std::string`.

```cpp
#include "LightHealth.h"
```

Imports the real implementation class and its generated `BnLightHealth` inheritance.

## Implementation-class shorthand

```cpp
using ::aidl::vendor::samsung::microxr::lighthealth::LightHealth;
```

Allows the file to write `LightHealth` instead of its complete namespace-qualified name. It does not affect the Binder descriptor.

## Process entry point

```cpp
int main() {
```

Begins execution of the native HAL binary started by Android init.

## Configure the Binder pool

```cpp
ABinderProcess_setThreadPoolMaxThreadCount(2);
```

Configures the process's NDK Binder pool for up to two Binder worker threads. These threads receive short AIDL operations such as trigger, registration, and unregistration.

```cpp
ABinderProcess_startThreadPool();
```

Starts Binder transaction processing before the service is published.

The Binder pool is separate from the worker inside `LightHealth`:

```text
Binder pool threads
    receive and dispatch incoming IPC calls

LightHealth::worker_
    runs the long LED/ALS health-check algorithm
```

## Construct the service object

```cpp
auto service = ndk::SharedRefBase::make<LightHealth>();
```

Allocates one `LightHealth` object on the HAL process heap and returns shared ownership of it. Its constructor also starts `LightHealth::worker_` and creates the listener death recipient.

## Build the service-manager name

```cpp
const std::string instance =
    std::string(LightHealth::descriptor) + "/default";
```

`LightHealth::descriptor` comes from the generated AIDL hierarchy:

```text
vendor.google.microxr.lighthealth.ILightHealth
```

Appending `/default` creates:

```text
vendor.google.microxr.lighthealth.ILightHealth/default
```

## Publish the service

```cpp
CHECK_EQ(
    AServiceManager_addService(
        service->asBinder().get(),
        instance.c_str()),
    STATUS_OK);
```

The expression performs these steps:

```text
LightHealth C++ object
        |
        | asBinder()
        v
Strong NDK Binder wrapper
        |
        | get()
        v
Raw AIBinder* expected by C API
        |
        | AServiceManager_addService(name)
        v
Service Manager stores name -> Binder reference
```

`instance.c_str()` supplies the C-string expected by the C API. Registration success returns `STATUS_OK`. If another status is returned, `CHECK_EQ` logs a fatal failure and terminates the process rather than leaving an unregistered HAL running.

## Keep the process serving Binder calls

```cpp
ABinderProcess_joinThreadPool();
```

Makes the calling thread participate in Binder processing and normally blocks for the process lifetime.

```cpp
return EXIT_FAILURE;
```

The pool is not expected to return during normal operation. If it does, `main()` reports process failure.

## Complete startup diagram

```text
Android init
    |
    | execute /vendor/bin/hw/vendor.samsung...service.default
    v
main()
    |
    +--> set Binder pool maximum to 2
    |
    +--> start Binder pool
    |
    +--> SharedRefBase::make<LightHealth>()
    |       |
    |       +--> construct BnLightHealth implementation
    |       `--> start dedicated health-check worker
    |
    +--> descriptor + "/default"
    |
    +--> asBinder() -> AIBinder*
    |
    +--> addService(name, Binder reference)
    |
    +--> require STATUS_OK
    |
    `--> join Binder pool and remain alive
```

After publication, an incoming call flows as follows:

```text
Client BpLightHealth
        |
        v
Kernel Binder driver
        |
        v
HAL Binder-pool thread
        |
        v
Generated BnLightHealth dispatch
        |
        v
LightHealth override
        |
        v
Dedicated worker for long check
```

---

# 17. Binder shared-reference ownership

## Simple definition

Shared-reference ownership means:

> The C++ service object remains alive while at least one shared owner still refers to it. When the last shared owner is released, the object is destroyed automatically.

This line returns shared ownership:

```cpp
auto service = ndk::SharedRefBase::make<LightHealth>();
```

The type is effectively:

```cpp
std::shared_ptr<LightHealth>
```

`SharedRefBase` is the NDK Binder utility that lets generated native Binder interface objects participate safely in this C++ shared-lifetime model.

## Reference-count example

```cpp
auto first = ndk::SharedRefBase::make<LightHealth>();
```

```text
Strong shared owners: 1

first
  |
  v
+-----------------------+
| LightHealth object    |
+-----------------------+
```

Copying the shared pointer adds an owner:

```cpp
auto second = first;
```

```text
Strong shared owners: 2

first  --------+
               |
               v
        +-----------------------+
        | One LightHealth object|
        +-----------------------+
               ^
               |
second --------+
```

Releasing one owner does not destroy the object:

```cpp
second.reset();
```

```text
Strong shared owners: 1

first
  |
  v
LightHealth remains alive
```

Releasing the last owner destroys it:

```cpp
first.reset();
```

```text
Strong shared owners: 0
        |
        v
LightHealth::~LightHealth()
        |
        +--> set stopping_=true
        +--> wake worker
        +--> join worker
        `--> release object memory
```

## Why not use a raw pointer?

A raw allocation requires manual deletion:

```cpp
LightHealth* service = new LightHealth();
```

Deleting too early can leave Binder trying to dispatch to invalid memory:

```text
Register raw object
      |
Delete object too early
      |
Client sends transaction
      |
Binder dispatch reaches deleted object
      |
Crash / use-after-free
```

Never deleting it causes a memory/resource leak. Shared ownership automatically chooses destruction when the final owner disappears.

A stack object has a similarly rigid scope-based lifetime and is not the normal NDK Binder service pattern:

```cpp
LightHealth service;  // destroyed whenever its stack scope ends
```

The normal pattern is:

```cpp
auto service = ndk::SharedRefBase::make<LightHealth>();
```

## Local C++ ownership versus remote Binder references

Two related but different reference systems exist:

```text
Inside HAL process:
    std::shared_ptr / SharedRefBase
    controls lifetime of the real C++ implementation

Across processes:
    Binder nodes, handles, and strong Binder references
    are tracked by Binder runtime and kernel driver
```

A remote client never receives a `std::shared_ptr<LightHealth>` because a C++ memory pointer is meaningful only in the HAL process. The client receives a Binder handle and owns a generated proxy.

```text
HAL PROCESS                         CLIENT PROCESS

shared_ptr<LightHealth>             shared_ptr<ILightHealth>
          |                                  |
          v                                  v
+----------------------+           +----------------------+
| Real LightHealth     |           | BpLightHealth proxy  |
+----------+-----------+           +----------+-----------+
           |                                  |
           v                                  v
    Server AIBinder <------ Binder driver --- client handle
```

The NDK Binder glue associates the local service implementation with its server Binder object. The kernel routes transactions using Binder references; the local shared ownership ensures the implementation is not accidentally destroyed while local owners still need it.

## Lifetime in this specific `main()`

The `service` variable is local to `main()`:

```cpp
auto service = ndk::SharedRefBase::make<LightHealth>();
```

Later, `main()` blocks in:

```cpp
ABinderProcess_joinThreadPool();
```

Therefore, the local `service` shared pointer normally remains in scope for the entire process lifetime:

```text
main enters
   |
   +--> create shared service owner
   |
   +--> register Binder object
   |
   `--> joinThreadPool()
            |
            | normally blocks forever
            v
       local owner remains alive
```

If the pool unexpectedly returns and `main()` exits, the local shared owner is released. When no shared owner remains, `LightHealth` is destroyed and its worker cleanup runs.

## Ownership summary

```text
Shared owner exists
    -> LightHealth remains alive

Shared pointer is copied
    -> another owner refers to the same object

One owner disappears
    -> object remains alive if another owner remains

Last owner disappears
    -> destructor runs automatically
```

This avoids both major raw-pointer lifetime errors:

```text
delete too early -> invalid Binder target / crash
never delete     -> memory and resource leak
```

---

# 18. Why AIDL methods return `ndk::ScopedAStatus`

The declaration:

```cpp
ndk::ScopedAStatus triggerHealthCheck() override;
```

contains two different concepts:

```text
ScopedAStatus
    = Did the Binder method call succeed?

LightHealthStatus
    = Was the device HEALTHY, TAMPERED, or UNKNOWN?
```

An AIDL method that conceptually returns `void` still needs to report IPC success, failure, exceptions, or service errors. The NDK backend therefore generates a C++ method returning `ndk::ScopedAStatus`.

The implementation is:

```cpp
ScopedAStatus LightHealth::triggerHealthCheck() {
  requestHealthCheck();
  return ScopedAStatus::ok();
}
```

`ok()` means the request was accepted and scheduled. It does not mean the hardware result is healthy. The hardware result arrives later through `onLightHealth()`.

```text
Client                         HAL
  |                             |
  | triggerHealthCheck()        |
  +---------------------------->|
  |                             | schedule worker
  |<----------------------------+
  | ScopedAStatus::ok()         |
  |                             |
  |              later         |
  |<----------------------------+
  | onLightHealth(HEALTHY/etc.) |
```

Common ways to construct the status are:

```cpp
ScopedAStatus::ok()
```

for successful method execution,

```cpp
ScopedAStatus::fromExceptionCode(EX_ILLEGAL_ARGUMENT)
```

for an AIDL exception, and

```cpp
ScopedAStatus::fromStatus(native_binder_status)
```

for a native Binder error. The generated stub serializes the status into a synchronous Binder reply, and the generated client proxy reconstructs it for the caller.

`Scoped` means the C++ wrapper automatically releases its underlying native status resource when it leaves scope.

---

# 19. Listener registration and unregistration

An `ILightHealthListener` is a Binder callback object implemented by the client. It allows the HAL to send the eventual health result in the reverse direction.

```text
Normal request:
Client ------------------------> LightHealth HAL
       triggerHealthCheck()

Callback:
Client listener <-------------- LightHealth HAL
                onLightHealth(status)
```

## Register

```cpp
registerHealthListener(listener)
```

means:

> Remember this listener and send it the current and future health results.

Registration performs:

```text
1. Reject null
2. Obtain listener Binder identity
3. Avoid duplicate entries
4. Create death-monitoring context
5. Link remote Binder death notification when applicable
6. Store the listener
7. Store its death cookie when linked
8. Copy current_status_
9. Unlock listener state
10. Immediately call onLightHealth(current_status_)
```

The listener is stored in:

```cpp
std::vector<std::shared_ptr<ILightHealthListener>> listeners_;
```

Several clients can register:

```text
listeners_
+---------------------+
| Client A listener   |
| Client B listener   |
| Client C listener   |
+---------------------+
```

A new listener receives the latest status immediately. Before any completed check, that value is `UNKNOWN`.

## Unregister

```cpp
unregisterHealthListener(listener)
```

means:

> Stop sending future health results to this listener.

Unregistration:

```text
1. Finds the listener by underlying AIBinder identity
2. Finds its death cookie
3. Unlinks Binder death monitoring
4. Deletes the cookie
5. Removes the listener
```

Calling unregister for an entry that is already absent still returns success, making the operation idempotent.

## Client-side callback implementation

A native client typically implements the generated listener server stub:

```cpp
class MyListener final : public BnLightHealthListener {
 public:
  ndk::ScopedAStatus onLightHealth(
      LightHealthStatus status) override {
    // Handle HEALTHY, TAMPERED, or UNKNOWN.
    return ndk::ScopedAStatus::ok();
  }
};
```

Then:

```cpp
auto listener = ndk::SharedRefBase::make<MyListener>();
service->registerHealthListener(listener);
service->triggerHealthCheck();
// Later: listener->onLightHealth(status) is dispatched in the client.
service->unregisterHealthListener(listener);
```

The actual client C++ object remains in the client process. The HAL stores a Binder interface/proxy that refers to it.

---

# 20. Death recipient, death cookie, and `AIBinder_linkToDeath`

## Death recipient

A death recipient is a callback registration object that receives notification when a remote Binder endpoint disappears, usually because its owning process died.

```text
death
    = remote Binder endpoint/process is gone

recipient
    = object that receives the notification
```

It does not cause a process to die. It only observes the event.

The service creates one recipient object:

```cpp
death_recipient_(
    AIBinder_DeathRecipient_new(
        &LightHealth::onListenerDied))
```

This stores the callback function:

```cpp
LightHealth::onListenerDied
```

## Death cookie

A death cookie is a per-listener identification/context tag. It is not a browser cookie or security token.

```cpp
struct ListenerCookie {
  LightHealth* service;
  AIBinder* binder;
};
```

It says:

```text
service -> which LightHealth owns the listener list
binder  -> which listener Binder should be removed
```

The callback is static and therefore has no automatic `this` pointer. The cookie supplies the service pointer and dead-listener identity it needs.

## `AIBinder_linkToDeath` arguments

```cpp
AIBinder_linkToDeath(
    binder,
    death_recipient_.get(),
    cookie.get());
```

means:

```text
binder
    = WHO to monitor: the client's listener Binder

death_recipient_.get()
    = WHAT callback object to notify

cookie.get()
    = WHAT per-listener information to pass to the callback
```

In plain language:

> Watch this listener Binder. If it dies, invoke `onListenerDied()` and pass it this listener's cookie.

`Binder` treats the `void*` cookie as opaque. It does not inspect the structure; it gives the same pointer back to the callback.

The `.get()` calls provide raw pointers required by the C API without transferring ownership. The scoped recipient still owns the recipient object, and the `unique_ptr` still owns the cookie.

## Cookie lifetime

After successful linkage, ownership is moved into:

```cpp
listener_cookies_.push_back(std::move(cookie));
```

This keeps the cookie memory alive for as long as Binder may return its pointer.

```text
Register
   |
   +--> create cookie
   +--> linkToDeath(..., cookie pointer)
   `--> store cookie in listener_cookies_
              |
              +------------------------------+
              |                              |
              v                              v
       Explicit unregister             Client process dies
              |                              |
       unlinkToDeath()                 onListenerDied(cookie)
              |                              |
       erase cookie                    remove listener
                                             |
                                      erase cookie
```

If `AIBinder_linkToDeath()` returns `STATUS_OK`, death monitoring is installed and the cookie is stored. `STATUS_INVALID_OPERATION` is accepted because death monitoring may not apply to a local Binder in the same process; in that case the listener is stored without a death cookie.

Death monitoring is needed because a client may crash or be killed without calling `unregisterHealthListener()`.

---

# 21. Binder-pool tasks versus dedicated-worker tasks

The HAL has a Binder pool and one ordinary `std::thread` worker. They serve different purposes.

## Binder-pool work

Incoming Binder transactions are decoded by generated `Bn` stubs and execute on Binder-pool threads:

```text
- triggerHealthCheck()
- requestHealthCheck() when called by trigger
- registerHealthListener()
- initial onLightHealth(current_status) call made during registration
- unregisterHealthListener()
- Binder death callback context
- incoming Light Behavior completion callback
```

`triggerHealthCheck()` only locks worker state, sets a pending flag, wakes the worker, and returns. It does not perform hardware work.

## Dedicated-worker work

The worker executes:

```text
- workerLoop() and condition-variable waiting
- runHealthCheck()
- injected CheckFunction when present
- core API-version validation
- LightHealthDeviceIo construction/initialization
- outgoing Light Behaviors Binder calls
- ALS polling and reads
- microxr_light_health_core_run()
- phase timing and sleeps
- final LED OFF
- DeviceIo cleanup
- notifyListeners() for completed results
```

Completed-result callbacks originate from the dedicated worker because `notifyListeners()` is called there. The remote listener implementation executes on a Binder thread in the client process.

```text
CLIENT                    HAL BINDER THREAD             HAL WORKER
  |                              |                          |
  | triggerHealthCheck()         |                          |
  +----------------------------->|                          |
  |                              | set requested flag       |
  |                              +------------------------->|
  |<-----------------------------+ return OK                |
  |                                                         |
  |                                      run hardware check  |
  |                                                         |
  |<--------------------------------------------------------+
  |                         onLightHealth(status)            |
```

The main service thread configures the pool, constructs/registers the service, then joins the pool and participates in Binder processing.

---

# 22. Why a mutex is needed with only one worker

The mutex is not protecting the worker from a second worker. It protects flags shared between different thread types:

```text
Binder thread
    writes check_requested_
    reads check_running_

Dedicated worker
    reads/writes check_requested_
    writes check_running_
    reads stopping_

Destruction thread
    writes stopping_
```

Protected state is:

```cpp
stopping_
check_requested_
check_running_
```

Without the mutex, simultaneous reads/writes would be a C++ data race. The mutex also works with the condition variable so checking the predicate and going to sleep do not miss a request notification.

```text
One worker thread does not mean one total thread.
The worker communicates with Binder threads and shutdown code.
```

---

# 23. Why use a dedicated worker and only two Binder threads?

## Why not run the complete check on Binder threads?

The health check performs slow operations:

```text
- service connection
- LED Binder calls
- ALS event polling
- three timed phases per blink
- repeated blink cycles
- core computation and cleanup
```

Running that directly inside `triggerHealthCheck()` would occupy a Binder thread for several seconds. With enough client requests, the Binder pool could be exhausted and registration, unregistration, callbacks, and death handling would be delayed.

```text
Binder thread -> schedule quickly -> return
Worker thread -> perform slow serialized hardware operation
```

Adding many Binder threads only postpones exhaustion and could allow conflicting concurrent LED/ALS checks. Binder threads are IPC dispatch resources, not a general-purpose hardware job queue. The single worker also provides simple ordering and coalescing.

## Why two Binder threads?

The AIDL entry points are intentionally short. Two threads allow useful overlap when one thread is briefly blocked, for example during the immediate listener callback:

```text
Binder thread 1 -> one incoming request/callback
Binder thread 2 -> another request remains serviceable
Worker thread   -> long hardware operation
```

`2` is a tuning choice, not a universal rule.

## How to choose a Binder-pool size

Consider:

```text
1. Peak number of simultaneous incoming calls
2. Typical and worst-case method duration
3. Synchronous outgoing Binder calls made inside an incoming method
4. Reentrant callbacks and death notifications
5. Required latency
6. CPU, stack-memory, scheduling, and lock-contention costs
```

General starting point:

```text
All entry methods short/nonblocking    -> 1-2 threads
Some brief synchronous blocking       -> 2-4 threads
High measured concurrency             -> tune using tracing/load tests
```

Increasing the Binder count does not speed up the intentionally serialized health-check worker.

---

# 24. Synchronous versus `oneway` Binder behavior

A normal AIDL method is synchronous unless the AIDL uses `oneway`.

## Synchronous transaction

```text
CLIENT THREAD              BINDER DRIVER              SERVER BINDER THREAD
     |                          |                              |
     | request                  |                              |
     +------------------------->+----------------------------->|
     |                    caller waits                         | execute
     |                          |                              |
     |<-------------------------+<-----------------------------+
     | reply/status                                            |
```

The client thread waits for the remote method reply. The reply can carry return data, `ScopedAStatus`, an exception, or an error.

## `oneway` transaction

```text
CLIENT THREAD              BINDER DRIVER              SERVER BINDER THREAD
     |                          |                              |
     | queue transaction       |                              |
     +------------------------->|                              |
     | returns without waiting |                              |
                                |----------------------------->| execute later
                                |                              | no normal reply
```

`oneway` does not mean no server Binder thread is used. It means the caller does not wait for remote completion. One-way traffic can still queue and experience backpressure.

## The trigger's application-level asynchronous pattern

`triggerHealthCheck()` is normally a short synchronous Binder call that schedules asynchronous local work:

```text
Synchronous Binder acceptance
    +
Asynchronous dedicated-worker execution
```

The client waits only until the HAL has accepted the trigger, not until the LED/ALS operation is complete.

Whether `onLightHealth()` and `setLightBehavior()` are Binder-level `oneway` methods must be confirmed from their `.aidl` definitions, which are not present in this workspace. Without `oneway`, those outgoing calls are synchronous and the calling HAL thread waits for the remote reply.

---

# 25. When a client needs its own Binder pool

A process that only makes outgoing synchronous calls and receives ordinary replies on its calling threads may not need to host a Binder thread pool.

A client that passes a Binder listener becomes a callback server and needs Binder threads to receive `onLightHealth()`:

```text
Client of ILightHealth
    -> sends trigger/register/unregister

Server for ILightHealthListener
    -> receives onLightHealth callbacks
```

Callback path:

```text
HAL worker
    |
    | onLightHealth(status)
    v
Binder driver
    |
    v
Client Binder-pool thread
    |
    v
BnLightHealthListener dispatch
    |
    v
MyListener::onLightHealth(status)
```

A standalone native client can start a pool using:

```cpp
ABinderProcess_setThreadPoolMaxThreadCount(2);
ABinderProcess_startThreadPool();
```

and must keep both its listener and process alive. It may join the pool if its main thread has no other event loop. Android Java application processes normally have Binder callback threads managed by the Android runtime and do not manually start them.

Short rule:

```text
Only outgoing calls, no callbacks/services
    -> a separate client Binder pool may not be needed

Receives listener callbacks or exposes Binder methods
    -> Binder processing threads are needed
```

---

# 26. Where Binder threads and objects reside

Binder threads, generated proxies/stubs, and service implementation objects live in user-space processes. They do not live inside the kernel Binder driver.

```text
CLIENT PROCESS
+----------------------------------+
| BpLightHealth proxy              |
| BnLightHealthListener callback   |
| Client Binder threads            |
+----------------+-----------------+
                 |
                 v
KERNEL BINDER DRIVER
+----------------------------------+
| Binder nodes and references      |
| Per-process handles              |
| Transactions and queues          |
| Routing and wakeups              |
+----------------+-----------------+
                 |
                 v
HAL PROCESS
+----------------------------------+
| LightHealth implementation       |
| BnLightHealth server stub        |
| HAL Binder-pool threads          |
| Dedicated health-check worker    |
+----------------------------------+
```

Service Manager is another user-space process. It stores:

```text
service name -> Binder reference
```

It does not contain or execute the real `LightHealth` object.

Final location summary:

```text
Binder threads and C++ objects  -> user processes
Service name mapping            -> Service Manager process
Handles, nodes, queues, routing  -> kernel Binder driver
```
