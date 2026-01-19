# SISEC Scraper Implementation Guide 

## 📋 Overview
This document provides step-by-step instructions for implementing the SISEC scraper integration end-to-end with G_IMSS. **You must study the existing IAD and MASTER scraper implementations and adapt their patterns for SISEC.**

## ⚠️ Important Instructions
- **DO NOT copy code from this document** - there is none provided
- **Study G_IAD.py and G_MASTER.py** to understand the patterns
- **Reference ScraperRouter.py** to see how other scrapers are integrated
- **Ask questions** if you don't understand the existing code
- **Test each component** before moving to the next step

---

## 🎯 Task 1: Integrate SISEC with G_IMSS.py

### Background
The G_IMSS.py file is the main orchestrator that:
1. Fetches NBC documents from MongoDB (Nueva_Base_Central collection)
2. Distributes tasks across different scrapers (IAD, IAD2, MASTER, SISEC)
3. Manages device assignment and queue creation
4. Monitors task execution and handles retries

### Implementation Steps

#### Step 1.1: Add SISEC Queue Collection Reference
**Location:** Inside `Generator.__init__()` method (around line 940)

**What you need to do:**
- Add a new queue collection reference for SISEC
- The collection should be named `imss_queue_sisec` in the `Mini_Base_Central` database

**Reference:** Look at how `collection_queue_iad` and `collection_queue_master` are declared in the same `__init__` method.

**Pattern to follow:** Follow the exact same pattern as IAD and MASTER queue collections.

---

#### Step 1.2: Create `insert_to_sisec_queue()` Method
**Location:** Inside `Generator` class (around line 1130, after `insert_to_master_queue()`)

**What you need to do:**
Create a new method that inserts a task into the SISEC queue. This method should:
1. Generate a unique RW (Registro Work) ID using the NBC document ID and timestamp
2. Extract CURP from the NBC document
3. Set CURP context for logging (using `set_curp_context()`)
4. Insert a document into the `collection_queue_sisec` with all necessary fields
5. Log the task assignment using `imss_logger`
6. Create a Registro Work document by calling `createRegistroWorkDocument()`
7. Update the NBC document with status 'p1' and the new RW ID
8. Clear the CURP context
9. Return True

**Reference:** Study `insert_to_iad_queue()` and `insert_to_master_queue()` methods in G_IMSS.py. Your SISEC version should follow the exact same pattern.

**Key differences for SISEC:**
- Handler name: `'IMSS_Sisec_DFL'`
- Instance name: `'SISEC_DFL_01'` (or dynamic based on configuration)
- Collection: `self.collection_queue_sisec` (not queue_iad or queue_master)

**Fields needed in queue document:**
- nbc_id, curp, task_data, rw_id, device, device_name, created_at, Status, success, Message, taskid

**Hint:** The method signature should accept: `nbc_doc`, `taskid`, and `device` as parameters.

---

#### Step 1.3: Update Main Processing Loop
**Location:** Inside `Generator.inicio()` method, main while loop (around line 1380)

**What you need to do:**
Add a conditional branch to handle when SmartDistributor chooses SISEC for task assignment.

**Reference:** Look for the section where you see:
- `elif chosen_generator == "G_IAD" and device_or_worker:`
- `elif chosen_generator == "G_MASTER" and device_or_worker:`

**What to add:** An additional `elif` statement for `"G_SISEC"` that calls your `insert_to_sisec_queue()` method.

**Pattern:** Must match the existing pattern exactly - check for both the generator name AND that device_or_worker is not None.

---

#### Step 1.4: Initialize G_SISEC Worker
**Location:** Inside `Generator.inicio()` method, at the start (around line 1220)

**What you need to do:**
1. Import the G_SISEC class at the top of the file
2. Create an instance of G_SISEC in the `inicio()` method
3. Start it as a background async task

**Reference:** Look at how `G_IAD` and `G_MASTER` are:
- Imported: `from G_IAD import G_IAD`
- Initialized: `self.g_iad = G_IAD()`
- Started: `iad_task = asyncio.create_task(self.g_iad.start_processing())`

**Follow the exact same pattern for G_SISEC.**

**Important:** The task must be started BEFORE the main processing loop begins, alongside iad_task and master_task.

---

#### Step 1.5: Update SmartDistributor for SISEC
**Location:** `SmartDistributor` class (around line 430)

**What you need to do:**
Update the SmartDistributor class in FIVE places:

**1. Update `__init__` method:**
- Add "sisec" to the `current_batch_assignments` dictionary (initialize to 0)
- Add "sisec" to the `batch_limits` dictionary (initialize to 0)  
- Add a reference to `dev_devices_collection` (SISEC uses a different collection than IAD/MASTER)

**Reference:** See how IAD, IAD2, and MASTER are initialized in the dictionaries.

**2. Update `initialize_batch_limits()` method:**
- Modify the unpacking to receive 4 values instead of 3 from `calculate_distribution_slots()`
- Update both dictionaries to include "sisec" with the calculated slots
- Reset the assignment counter

**Reference:** Look at the existing implementation and add SISEC as the 4th element.

**3. Update `assign_single_task()` method - Add availability check:**
- Add a check for SISEC device availability (similar to IAD and MASTER checks)
- Query the `dev_devices_collection` (not `devices_collection`) for SISEC devices
- Check if SISEC is enabled in config
- Check if batch limit not exceeded
- Add "sisec" to `available_options` if devices are available

**Reference:** Study the IAD and MASTER availability checks in the same method.

**Important:** SISEC devices are in `dev_devices` collection with `process: "SISEC"`.

**4. Update rotation logic:**
- Change from modulo 3 to modulo 4 rotation
- Add SISEC as the 4th option in rotation: IAD → MASTER → IAD2 → SISEC
- Assign when `assignment_counter % 4 == 3`

**Reference:** Look at the existing modulo 3 rotation pattern and extend it.

**5. Add SISEC device assignment logic:**
- Add an `elif chosen_option == "sisec":` branch
- Call `get_and_assign_device_atomic_sisec()` method
- Increment assignment counter and batch assignments
- Return the tuple `("G_SISEC", device)` on success

**Reference:** Study the IAD and MASTER assignment branches - follow the same pattern.

---

#### Step 1.6: Update DeviceManager for SISEC Devices
**Location:** `DeviceManager` class (around line 150)

**What you need to do:**
Add THREE new methods to DeviceManager:

**Method 1: `cleanup_stuck_dev_devices()`**
- Purpose: Clean up SISEC devices that have been stuck processing for too long
- Should query `dev_devices` collection (not regular `devices`)
- Find devices with `available: False` and old `task_start_time`
- Release them by calling `release_dev_device()`

**Reference:** Study `cleanup_stuck_devices()` method - follow the exact same logic but use `dev_devices_collection`.

**Method 2: `get_and_assign_device_atomic_sisec()`**
- Purpose: Atomically find and assign an available SISEC device
- Parameters: process_type, taskid, curp
- Must prevent duplicate CURP assignments (check last 18 minutes)
- Clean up stale assignments for the same CURP
- Use `find_one_and_update()` for atomic assignment
- Mark device as unavailable and set current_task, task_id, task_start_time

**Reference:** Study `get_and_assign_device_atomic()` - this method should be nearly identical but use `dev_devices` collection.

**Important differences:**
- Query `dev_devices_collection` instead of `devices_collection`
- Same atomic pattern with duplicate prevention
- Same cleanup of stale tasks

**Method 3: `release_dev_device()`**
- Purpose: Release a SISEC device after task completion or failure
- Parameters: device_id, taskid
- Should update `dev_devices` collection
- Set `available: True` and update timestamp
- Unset current_task, task_id, task_start_time

**Reference:** Study `release_device()` method - adapt it for `dev_devices` collection.

**Key point:** All three methods work with `dev_devices` collection, not the regular `devices` collection.

---

#### Step 1.7: Update PerformanceMonitor for SISEC
**Location:** `PerformanceMonitor` class

**What you need to do:**
Modify the `calculate_distribution_slots()` method:

1. **Change return values:** Method must now return 4 values instead of 3 (add sisec_slots)

2. **Get SISEC metrics:** Call `get_generator_metrics("G_SISEC")` to get success rate

3. **Calculate SISEC slots based on performance:**
   - If success_rate >= high threshold: use distribution_good array index 3
   - If success_rate >= low threshold: use distribution_normal array index 3
   - Otherwise: use distribution_bad array index 3
   - Convert percentage to actual slot count

4. **Return:** `return iad2_slots, iad_slots, master_slots, sisec_slots`

**Reference:** Look at how IAD2, IAD, and MASTER slots are calculated in the same method. Add SISEC as the 4th calculation using the same pattern.

**Important:** The distribution arrays in config must have 4 elements: [IAD2_%, IAD_%, MASTER_%, SISEC_%]

---

#### Step 1.8: Update Configuration
**Location:** `ConfigurationManager` class `__init__` method

**What you need to do:**
Update the imss_config collection document:

1. **Expand distribution arrays to 4 elements:**
   - `distribution_good`: Add 4th value for SISEC percentage

2. **Add SISEC enable flag:**
   - Add `"sisec_enabled": True` to the document

**Reference:** Look at the existing default_config dictionary. See how `iad_enabled`, `iad2_enabled`, `master_enabled` are set.

**Recommendation for distribution percentages:**
- All percentages in each array should sum to 100
- Start conservative: give SISEC smaller percentage initially after that we will increase it.
- Example: `distribution_good: [40, 25, 25, 10]` (5% to SISEC when performing well)

**Arrays represent:** 
[MASTER_%, "_", IAD2_%, SISEC_%]
IAD% = BATCH_SIZE - (MASTER_% + IAD2_% + SISEC_%)

---

## 🎯 Task 2: Create G_SISEC.py

### Purpose
G_SISEC.py is a standalone worker that:
1. Continuously monitors the `imss_queue_sisec` collection
2. Picks up tasks with Status='Assigned'
3. Calls the SISEC scraper through ScraperRouter
4. Handles responses and updates MongoDB documents
5. Manages device lifecycle

### Reference Files
- **G_IAD.py** - Primary reference for queue-based processing
- **G_MASTER.py** - Secondary reference for alternative patterns

### Implementation Steps

#### Step 2.1: Create File Structure
Create a new file: `G_SISEC.py` in the same directory as G_IAD.py and G_MASTER.py

---

#### Step 2.2: Add Imports and Logger Setup
**What you need to do:**
Set up all necessary imports and configure logging (both console and MongoDB logging).

**Reference:** Look at the import section of G_IAD.py (lines 1-25). You need:
- Standard Python libraries: logging, asyncio, traceback
- MongoDB libraries: ObjectId from bson
- Date/time: datetime, timedelta
- Project imports: MongoConnection, ScraperRouter
- Logger imports: get_imss_logger, set_curp_context, clear_curp_context

**Logging setup:**
- Configure basicConfig for console/file logging
- Get standard logger: `logger = logging.getLogger(__name__)`
- Get IMSS logger for MongoDB: `imss_logger = get_imss_logger(__name__)`

**Why two loggers?**
- `logger` = console/file logs for debugging
- `imss_logger` = MongoDB logs for tracking tasks by CURP

---

#### Step 2.3: Create G_SISEC Class and __init__
**What you need to do:**
Create the main class and initialize MongoDB collection references.

**Reference:** Study `G_IAD.__init__()` method (G_IAD.py, lines 26-32).

**Collections you need:**
1. `collection_NBC` - Nueva_Base_Central (Main_User database)
2. `collection_queue_sisec` - imss_queue_sisec (Mini_Base_Central database)
3. `collection_generators` - imss_generators (Mini_Base_Central database)
4. `collection_devices` - **IMPORTANT:** Use `dev_devices` collection (not regular `devices`)

**Key difference from IAD/MASTER:** SISEC uses `dev_devices` collection for device management.

**Pattern:** Follow the exact same structure as G_IAD, but change queue collection name to `imss_queue_sisec` and devices to `dev_devices`.

---

#### Step 2.4: Create `process_sisec_job()` Method
**What you need to do:**
Create the main job processing method that handles a single SISEC task.

**Reference:** Study `process_iad_job()` in G_IAD.py (lines 34-140). Your method should follow this exact structure.

**Method flow:**
1. **Extract data** from queue_doc (curp, device_id, device_name, taskid, nbc_id, task_data, queue_document_id)
2. **Set CURP context** for logging
3. **Log job start** 
4. **Try block:**
   - Extract name_data dictionary
   - Create ScraperRouter instance with handlerid and instance name
   - Call `scraper_router.scrape()` with all parameters
   - Log scraper result
   - Get updated RW ID from NBC
   - **Check result:**
     - If "assigned" in message and status=False → Job assigned successfully
     - Otherwise → Condition 4 failure
5. **Exception handling:** Update queue with error, release device
6. **Finally block:** Clear CURP context

**Parameters for ScraperRouter:**
- handlerid: `'IMSS_Sisec_DFL'`
- instance: `'IMSS_SISEC_DFL_01'`

**Condition 4 handling (when assignment fails):**
- Update Registro_works with Status='Complete', Condition=4
- Release device (using `dev_devices` collection)
- Update NBC for retry
- Update queue status to Complete/Failed

**Important:** Use `self.collection_devices` (which points to `dev_devices`) for device operations.

---

#### Step 2.5: Create `update_capacity_status()` Method
**What you need to do:**
Report SISEC capacity metrics to the generators collection so G_IMSS knows SISEC availability.

**Reference:** Study the same method in G_IAD.py or G_MASTER.py.

**What this method does:**
1. Count available SISEC devices (online=True, available=True, process='SISEC')
2. Count total online SISEC devices
3. Count current active tasks (Status='Assigned' in queue)
4. Update or insert document in `collection_generators` with _id='G_SISEC'

**Fields to update:**
- generator_name, instance, status, available_devices, total_devices
- available_capacity, current_active_tasks, last_update

**Error handling:** If exception occurs, mark status as 'fallando' (failing)

**Pattern:** Nearly identical to IAD, just change:
- Collection name to `dev_devices`
- Process filter to 'SISEC'
- Generator ID to 'G_SISEC'
- Queue collection to `collection_queue_sisec`

---

#### Step 2.6: Create `start_processing()` Method
**What you need to do:**
Create the main infinite loop that continuously monitors and processes the queue.

**Reference:** Study `start_processing()` in G_IAD.py (lines 175-220).

**Method structure:**
1. Log startup message
2. Infinite while loop:
   - Update capacity status
   - Find oldest task with Status='Assigned'
   - Atomically change status to 'Processing' using `find_one_and_update()`
   - If task found: process it
   - If no tasks: sleep 5 seconds
   - Exception handling: log error and sleep 10 seconds

**Important MongoDB operation:**
```
find_one_and_update(
    {'Status': 'Assigned'},
    {'$set': {'Status': 'Processing'}},
    sort=[('created_at', 1)],  # Oldest first
    return_document=True
)
```

**Why atomic update?** Prevents race conditions when multiple workers try to grab the same task.

---

#### Step 2.7: Add Main Entry Point
**What you need to do:**
Add the `if __name__ == '__main__':` block to allow standalone execution.

**Reference:** Look at the bottom of G_IAD.py or G_MASTER.py.

**Pattern:** Use asyncio to run the `start_processing()` method.

**Purpose:** Allows testing G_SISEC independently before integrating with G_IMSS.

---

## 🎯 Task 3: Update ScraperRouter.py

### Purpose
ScraperRouter acts as a gateway to external scraper APIs. It routes requests to different scraper endpoints based on the instance name.

### Reference Files
- **ScraperRouter.py** - Study existing scraper methods like `scrape_imss_app_priyansh()` and `scrape_imss_app_iad2()`

### Implementation Steps

#### Step 3.1: Add SISEC Scraper Method
**Location:** Inside `ScraperRouter` class (after existing scraper methods like `scrape_imss_app_iad2()`)

**What you need to do:**
Create a new async method called `scrape_imss_app_sisec()` that sends HTTP requests to the SISEC scraper API.

**Reference:** Study `scrape_imss_app_priyansh()` and `scrape_imss_app_iad2()` methods. Your SISEC method should follow the same pattern.

**Method structure:**
1. **Parameters:** curp, device_id, taskid, nbc_id (optional), queue_document_id (optional)
2. **Convert ObjectIds to strings** for JSON serialization
3. **Define endpoint mappings:**
   - Map SISEC endpoints (ngrok URLs) to instance names
   - Map device instances to specific endpoints
4. **Get device instance** from dev_devices collection
5. **Select correct endpoint** based on device instance
6. **Retry logic** with max_retries
7. **HTTP POST request** using aiohttp
8. **Handle response codes:**
   - 200: Success - task assigned
   - 429: Server busy
   - 400: Bad request
   - Other: Unknown error
9. **Return self.dictReturn**

**Endpoint mapping example structure:**
- Define a dictionary mapping URLs to instance names (e.g., "SISEC_OM_DFL_01")
- Define a dictionary mapping device instances to URLs (e.g., "SISEC-OM" → URL)
- Query dev_devices to get the device's instance field
- Use instance to select correct URL

**HTTP request JSON body should include:**
- curp
- nbc_id (as string)
- device_id
- taskid
- queue_document_id (as string)

**Important differences from IAD:**
- SISEC may have multiple endpoints (different servers)
- Must query `dev_devices` collection to get device instance
- Must support endpoint selection based on device instance

**Logging:** Use `imss_logger` for detailed logging of:
- API call preparation
- Endpoint selection
- Request/response details
- Success/failure outcomes

**Error handling:**
- Wrap in try/except
- Handle network errors
- Retry on failure
- Return appropriate error messages in dictReturn

---

#### Step 3.2: Update `scrape()` Method Routing
**Location:** Inside `ScraperRouter.scrape()` method (at the end, after existing instance checks)

**What you need to do:**
Add a new conditional branch that routes SISEC requests to the SISEC scraper method.

**Reference:** Look at existing instance checks in the `scrape()` method:
```
if self.instance_name == "IMSS_App_DFL_Priyansh_01":
    # IAD routing
if self.instance_name == "IMSS_App_DFL_Priyansh_02":
    # IAD2 routing
if self.instance_name == "IMSS_Master_DFL_01":
    # MASTER routing
```

**What to add:**
An additional `if` statement that:
1. Checks if `self.instance_name == "IMSS_SISEC_DFL_01"`
2. Logs the scraper being used
3. Calls `await self.scrape_imss_app_sisec()`
4. Returns `self.dictReturn`

**Pattern:** Follow the exact same pattern as IAD and IAD2 routing.

**Instance name to check:** `"IMSS_SISEC_DFL_01"` (must match what you set in G_SISEC.py)

**Log message format:** Include CURP, device_id, and taskid for tracking.

---

## 📊 MongoDB Collections Structure

### Collection: `imss_queue_sisec`
**Purpose:** Stores SISEC tasks waiting to be processed

**How to study:** Look at `imss_queue_iad` or `imss_queue_master` collections in MongoDB Compass to understand the structure.

**Document fields you need:**
- `_id` - MongoDB ObjectId (auto-generated)
- `nbc_id` - Reference to NBC document (ObjectId)
- `curp` - Person's CURP (string)
- `task_data` - Complete NBC document (object)
- `rw_id` - Registro Work ID (string, format: "IMSS_{nbc_id}_{timestamp}")
- `device` - Assigned device ID (string)
- `device_name` - Human-readable device name (string)
- `created_at` - Task creation timestamp (Date)
- `Status` - Task status (string: "Assigned", "Processing", "Complete")
- `success` - Task outcome (null, true, or false)
- `Message` - Error/success message (string)
- `taskid` - Sequential task ID (number)

**Reference:** Query an existing queue collection to see actual documents:
```javascript
db.imss_queue_iad.findOne()
```

---

### Collection: `dev_devices`
**Purpose:** Manages SISEC device availability and assignments

**How to study:** Look at the regular `devices` collection structure - `dev_devices` follows the same pattern but for SISEC devices.

**Document fields you need:**
- `_id` - MongoDB ObjectId
- `device` - Unique device ID (string, e.g., "SISEC-OM-001")
- `device_name` - Human-readable name (string)
- `process` - Process type (string, must be "SISEC")
- `instance` - Instance identifier (string, e.g., "SISEC-OM")
- `online` - Device online status (boolean)
- `available` - Currently available? (boolean)
- `current_task` - CURP being processed (string, unset when available)
- `task_id` - Task ID (number, unset when available)
- `task_start_time` - When task started (Date, unset when available)
- `updated_at` - Last update time (Date)

**Creating a test device:**
You'll need to insert a SISEC device document to test. Study how IAD devices are structured in the `devices` collection.

---

### Collection: `imss_generators`
**Purpose:** Tracks generator status and capacity

**How to study:** Query existing generator documents:
```javascript
db.imss_generators.find()
```

**Your SISEC generator document structure:**
- `_id` - Must be "G_SISEC" (string, not ObjectId)
- `generator_name` - "G_SISEC"
- `instance` - "SISEC"
- `status` - "running" or "fallando"
- `available_devices` - Count of available devices (number)
- `total_devices` - Count of total online devices (number)
- `available_capacity` - Tasks that can be accepted (number)
- `current_active_tasks` - Currently processing tasks (number)
- `last_update` - Last status update (Date)
- `last_failure_date` - Last failure timestamp (Date, optional)

**Reference:** Look at G_IAD or G_MASTER documents in this collection to understand the pattern.

---

## 🧪 Testing Checklist

### Pre-Testing Setup
- [ ] Create a test SISEC device in `dev_devices` collection
- [ ] Enable SISEC in configuration (`sisec_enabled: true`)
- [ ] Set up distribution percentages for SISEC
- [ ] Configure SISEC endpoint URL (ngrok or actual server)

### Unit Testing (Test Each Component Separately)

**Test `insert_to_sisec_queue()` Method:**
- [ ] Method creates queue document with all required fields
- [ ] RW ID is generated correctly (format: IMSS_{nbc_id}_{timestamp})
- [ ] CURP context is set and cleared properly
- [ ] Logs appear in both console and MongoDB
- [ ] Method returns True on success

**Test `process_sisec_job()` Method:**
- [ ] Method successfully calls ScraperRouter
- [ ] Handles 200 response (successful assignment) correctly
- [ ] Handles non-200 response (sets condition 4) correctly
- [ ] Releases device on failure
- [ ] Updates NBC, RW, and queue documents correctly
- [ ] Exception handling works (try/except/finally)

**Test Device Assignment:**
- [ ] `get_and_assign_device_atomic_sisec()` finds available devices
- [ ] Prevents duplicate CURP assignments
- [ ] Cleans up stale tasks correctly
- [ ] Returns None when no devices available
- [ ] Marks device as unavailable atomically

**Test Device Release:**
- [ ] `release_dev_device()` marks device as available
- [ ] Unsets current_task, task_id, task_start_time
- [ ] Updates timestamp correctly
- [ ] Handles task_id mismatch gracefully

### Integration Testing (Test Full Flow)

**Test Full SISEC Flow:**
- [ ] NBC document → Queue creation
- [ ] Queue → G_SISEC picks up task
- [ ] G_SISEC → ScraperRouter → External API
- [ ] Response handling → Database updates
- [ ] Device lifecycle (assign → process → release)

**Test SmartDistributor:**
- [ ] SISEC appears in rotation (every 4th task)
- [ ] Batch limits respected for SISEC
- [ ] Falls back to other scrapers when SISEC unavailable
- [ ] Available options calculated correctly

**Test Capacity Reporting:**
- [ ] `imss_generators` collection updates every cycle
- [ ] Device counts are accurate
- [ ] Active task counts match queue
- [ ] Status changes to "fallando" on errors

**Test Error Scenarios:**
- [ ] SISEC API returns 429 (server busy) → condition 4
- [ ] SISEC API returns 400 (bad request) → condition 4
- [ ] Network timeout → condition 4
- [ ] Device timeout → device released, task retried

### Performance Testing

**Monitor Under Load:**
- [ ] G_SISEC processes tasks without memory leaks
- [ ] Queue doesn't grow indefinitely
- [ ] Device assignment/release is timely
- [ ] MongoDB queries are efficient (use indexes)
- [ ] Logging doesn't cause performance issues

**Benchmarks to Track:**
- Average task processing time
- Device utilization rate (tasks/hour per device)
- Success rate percentage
- Failure rate and condition 4 percentage
- Queue wait time (time between creation and processing)

---

## 🐛 Common Issues and Solutions

### Issue 1: Tasks not being assigned to SISEC

**Symptoms:**
- No tasks appear in `imss_queue_sisec`
- SmartDistributor always chooses IAD/MASTER
- SISEC never appears in rotation

**Debugging steps:**
1. Check `imss_config` collection:
   ```javascript
   db.imss_config.findOne({_id: "main_config"})
   ```
   - Is `sisec_enabled: true`?
   - Do distribution arrays have 4 elements?

2. Check `dev_devices` collection:
   ```javascript
   db.dev_devices.find({process: "SISEC"})
   ```
   - Are there any SISEC devices?
   - Are any `online: true` and `available: true`?

3. Check `imss_generators` collection:
   ```javascript
   db.imss_generators.findOne({_id: "G_SISEC"})
   ```
   - Is status "running" (not "fallando")?
   - Is `available_capacity > 0`?

4. Check G_SISEC logs:
   - Is G_SISEC process running?
   - Any errors in startup?

**Common fixes:**
- Enable SISEC in config
- Add online SISEC devices
- Restart G_SISEC if status is "fallando"
- Check distribution percentages (shouldn't all be 0)

---

### Issue 2: Devices not releasing after task completion

**Symptoms:**
- All SISEC devices stuck in `available: false`
- `current_task` field never gets unset
- No new tasks can be assigned

**Debugging steps:**
1. Check stuck devices:
   ```javascript
   db.dev_devices.find({
       process: "SISEC",
       available: false
   })
   ```
   - How old is `task_start_time`?
   - Does `task_id` match a real task?

2. Check if `release_dev_device()` is being called:
   - Look for release logs in G_SISEC output
   - Check if task_id matches between queue and device

3. Check queue documents:
   ```javascript
   db.imss_queue_sisec.find({Status: "Processing"})
   ```
   - Are there stuck tasks in "Processing" status?

**Common fixes:**
- Verify `task_id` parameter matches in release call
- Check device timeout cleanup is running
- Manually release stuck devices:
  ```javascript
  db.dev_devices.update(
      {device: "SISEC-OM-001"},
      {$set: {available: true, updated_at: new Date()},
       $unset: {current_task: "", task_id: "", task_start_time: ""}}
  )
  ```
- Restart G_SISEC to trigger cleanup

---

### Issue 3: SISEC API not responding

**Symptoms:**
- All SISEC tasks get condition 4
- "Error in SISEC Scraper" messages
- Timeout errors in logs

**Debugging steps:**
1. Test endpoint directly:
   - Use Postman or curl to test the SISEC ngrok URL
   - Send a test POST request with sample data
   - Check if API is reachable

2. Check endpoint configuration:
   - Verify ngrok URL in `scrape_imss_app_sisec()` method
   - Ensure ngrok tunnel is active
   - Check if endpoint mapping is correct

3. Review ScraperRouter logs:
   - Look for connection errors
   - Check timeout settings (currently 120 seconds)
   - Verify JSON payload format

**Common fixes:**
- Restart ngrok tunnel
- Update ngrok URL in code
- Check network connectivity
- Verify API expects correct JSON structure
- Increase timeout if API is slow
- Check API server logs for errors

---

### Issue 4: Duplicate CURP processing

**Symptoms:**
- Same CURP assigned to multiple SISEC devices
- Conflicting updates to NBC documents
- Race condition errors

**Debugging steps:**
1. Check for duplicate assignments:
   ```javascript
   db.dev_devices.find({current_task: "ABCD123456..."})
   ```
   - Should only return 1 device per CURP

2. Review `get_and_assign_device_atomic_sisec()`:
   - Is duplicate check working? (18-minute window)
   - Is `find_one_and_update()` being used (atomic)?

**Common fixes:**
- Ensure atomic operations are used
- Check duplicate detection logic
- Verify 18-minute cleanup window
- Use transaction if needed

---

### Issue 5: Condition 4 for all SISEC tasks

**Symptoms:**
- Every SISEC task fails with condition 4
- No tasks complete successfully
- NBC documents all retry

**Debugging steps:**
1. Check API response:
   - What HTTP status code is the API returning?
   - Is the response format correct?

2. Review response handling logic in `process_sisec_job()`:
   - What message is in the result?
   - Does message contain "assigned"?
   - Is status False?

3. Test external scraper:
   - Is the SISEC scraper running?
   - Can it accept tasks?
   - Check its logs for errors

**Common fixes:**
- Verify external scraper is running
- Check response message format
- Ensure API returns proper status codes
- Review condition logic in G_SISEC

---

## 📝 Code Review Checklist

Before submitting your code for review, verify:

### Code Structure
- [ ] All files are in correct locations
- [ ] Imports are organized (standard library → third party → project)
- [ ] No unused imports
- [ ] Class and method names follow existing conventions
- [ ] File follows same structure as IAD/MASTER references

### Logging
- [ ] Both `logger` and `imss_logger` configured correctly
- [ ] CURP context set at start of processing
- [ ] CURP context cleared in finally blocks
- [ ] Log messages include useful context (taskid, device, CURP)
- [ ] Log levels appropriate (info, warning, error)
- [ ] No sensitive data logged (passwords, API keys)

### Error Handling
- [ ] All async operations wrapped in try/except
- [ ] Exception messages are descriptive
- [ ] Tracebacks logged on errors
- [ ] Resources released in finally blocks (context, devices)
- [ ] Graceful degradation (errors don't crash the service)

### MongoDB Operations
- [ ] Collection names match exactly (case-sensitive)
- [ ] Field names match existing conventions
- [ ] ObjectIds converted to strings when needed
- [ ] Atomic operations used where needed (`find_one_and_update`)
- [ ] Indexes considered for queried fields
- [ ] No hardcoded database/collection names where config exists

### Async/Await
- [ ] All MongoDB operations use `await`
- [ ] HTTP requests use `await`
- [ ] `asyncio.sleep()` used (not `time.sleep()`)
- [ ] No blocking operations in async functions
- [ ] Async context managers used correctly (`async with`)

### Device Management
- [ ] Devices assigned atomically
- [ ] Devices released on success AND failure
- [ ] Duplicate CURP detection working
- [ ] Stale task cleanup implemented
- [ ] Timeout handling in place
- [ ] Uses `dev_devices` collection (not `devices`)

### Configuration
- [ ] No hardcoded values that should be in config
- [ ] Config loaded from MongoDB
- [ ] Default values provided for missing config
- [ ] Distribution arrays have 4 elements
- [ ] Enable/disable flags respected

### Code Quality
- [ ] No commented-out code blocks
- [ ] No debug print statements
- [ ] Comments explain WHY, not WHAT
- [ ] Complex logic has explanatory comments
- [ ] Variable names are descriptive
- [ ] No magic numbers (use named constants or config)

### Testing Evidence
- [ ] Unit tests created and passed
- [ ] Integration test performed
- [ ] Test CURP processed successfully
- [ ] Error scenarios tested
- [ ] Screenshots or logs of successful test runs

### Documentation
- [ ] Docstrings added to all methods
- [ ] Complex sections have inline comments
- [ ] README updated if needed
- [ ] API endpoint documentation updated

---

## 🚀 Deployment Steps

### Step 1: Prepare MongoDB Collections

1. **Create indexes** (optional but recommended for performance):
   ```javascript
   // Index for faster queue processing
   db.imss_queue_sisec.createIndex({Status: 1, created_at: 1})
   
   // Index for device queries
   db.dev_devices.createIndex({process: 1, online: 1, available: 1})
   ```

2. **Insert test SISEC device:**
   ```javascript
   db.dev_devices.insert({
       device: "SISEC-OM-001",
       device_name: "SISEC Test Device 1",
       process: "SISEC",
       instance: "SISEC-OM",
       online: true,
       available: true,
       updated_at: new Date()
   })
   ```

3. **Verify configuration:**
   ```javascript
   db.imss_config.findOne({_id: "main_config"})
   ```
   - Check `sisec_enabled: true`
   - Check distribution arrays have 4 elements
   - Update if needed

---

### Step 2: Deploy Code Files

1. **Upload G_SISEC.py:**
   - Place in same directory as G_IAD.py and G_MASTER.py
   - Verify file permissions (should be executable)

2. **Update G_IMSS.py:**
   - Deploy your modified version
   - Create backup of current version first
   - Verify all changes are included

3. **Update ScraperRouter.py:**
   - Deploy modified version with SISEC methods
   - Backup current version first

4. **Test imports:**
   ```bash
   python -c "from G_SISEC import G_SISEC; print('Import successful')"
   ```

---

### Step 3: Start Services

1. **Start G_SISEC first (for testing):**
   ```bash
   python G_SISEC.py
   ```
   - Watch for startup message: "🚀 Starting SISEC processing..."
   - Check for errors in first 30 seconds
   - Verify capacity status updates in `imss_generators` collection

2. **Stop current G_IMSS** (if running):
   ```bash
   # Find process
   ps aux | grep G_IMSS.py
   
   # Stop gracefully
   kill <pid>
   ```

3. **Start new G_IMSS:**
   ```bash
   python G_IMSS.py
   ```
   - Watch for G_SISEC initialization
   - Check all 4 generators start successfully
   - Monitor first batch processing

---

### Step 4: Monitor Initial Operation

1. **Watch logs** for first few minutes:
   ```bash
   tail -f logs/imss.log | grep -E "SISEC|G_SISEC"
   ```

2. **Check MongoDB collections:**
   ```javascript
   // Check if queue is being populated
   db.imss_queue_sisec.find().count()
   
   // Check generator status
   db.imss_generators.findOne({_id: "G_SISEC"})
   
   // Check device status
   db.dev_devices.find({process: "SISEC"})
   ```

3. **Monitor first task:**
   - Watch for task assignment in SmartDistributor
   - Verify queue document created
   - Check G_SISEC picks up task
   - Verify API call to external scraper
   - Check response handling

---

### Step 5: Verify External Scraper Integration

1. **Ensure external SISEC scraper is running:**
   - Check ngrok tunnel is active
   - Verify endpoint URL is accessible
   - Test endpoint manually if needed

2. **Monitor first API interaction:**
   - Check ScraperRouter logs for HTTP request
   - Verify response status code
   - Check if task assigned successfully or condition 4

3. **Verify device lifecycle:**
   - Device assigned → unavailable
   - Task processed
   - Device released → available again

---

### Step 6: Production Monitoring

**Key metrics to track:**

1. **Task distribution:**
   ```javascript
   // Count tasks per generator today
   db.imss_queue_sisec.find({
       created_at: {$gte: new Date(new Date().setHours(0,0,0,0))}
   }).count()
   ```

2. **Success rate:**
   ```javascript
   // SISEC success rate
   db.imss_queue_sisec.aggregate([
       {$match: {Status: "Complete"}},
       {$group: {
           _id: "$success",
           count: {$sum: 1}
       }}
   ])
   ```

3. **Device utilization:**
   ```javascript
   // Check device usage
   db.dev_devices.aggregate([
       {$match: {process: "SISEC"}},
       {$group: {
           _id: "$available",
           count: {$sum: 1}
       }}
   ])
   ```

4. **Performance metrics:**
   - Average task completion time
   - Queue wait time
   - Condition 4 rate
   - API response times

**Set up alerts for:**
- All SISEC devices unavailable for > 10 minutes
- SISEC success rate < 50%
- Queue size > 100 tasks
- G_SISEC status = "fallando"

---

### Step 7: Gradual Rollout

**Day 1:**
- Monitor closely
- Keep SISEC percentage low (5%)
- Fix any immediate issues

**Day 2-3:**
- If stable, increase to 10-15%
- Monitor success rates
- Compare with IAD/MASTER performance

**Week 2:**
- Adjust distribution based on performance
- Add more SISEC devices if needed
- Optimize based on metrics

---

## 📚 Reference Documentation

### Existing Similar Files
- **G_IAD.py** - Reference for queue-based processing
- **G_MASTER.py** - Reference for device management
- **ScraperRouter.py** - Reference for API integration patterns

### Key Differences for SISEC
1. Uses `dev_devices` collection instead of `devices`
2. Has separate `release_dev_device()` method
3. Supports multiple endpoints with instance mapping
4. Different instance naming: `SISEC_DFL_01` vs `IMSS_App_DFL_Priyansh_01`

---

## ✅ Final Checklist

- [ ] Task 1: G_IMSS.py updated with SISEC integration
- [ ] Task 2: G_SISEC.py created with all methods
- [ ] Task 3: ScraperRouter.py updated with SISEC scraper
- [ ] MongoDB collections verified
- [ ] Configuration updated
- [ ] Code tested locally
- [ ] Logs reviewed for errors
- [ ] Code reviewed by senior developer
- [ ] Documentation updated
- [ ] Deployed to production

---
