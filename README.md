# Moodle Page Activity Modification: Video Completion and Attempt Tracking

This README documents modifications to the built-in Moodle "Page" activity module. These changes add the following features:
- Restrict video skip-forwarding to only allow seeking up to the furthest point watched.
- Track and store user attempts, including time watched and completion status.
- Add a custom completion rule that marks the activity as complete only when the entire video is watched.
- Introduce an "Attempts" page accessible via the activity navbar, allowing teachers and students to view all user attempts (e.g., number of attempts, time watched, completion status, and date).

These modifications involve creating new files, editing existing ones, adding database tables/columns, and defining language strings. They are designed for Moodle environments (tested with Moodle 4.x+).

## Prerequisites
- Access to the Moodle codebase (e.g., via FTP or server access).
- Database access (e.g., MySQL/MariaDB) to run SQL queries.
- Basic knowledge of PHP, JavaScript, and Moodle plugin development.

**Note:** These changes modify core Moodle files. Back up your site before applying them. Consider creating a custom plugin instead for production to avoid core hacks.

## Features
- **Video Skip Restriction:** Prevents users from skipping ahead in videos beyond what they've watched.
- **Attempt Logging:** Saves watch time and completion on page unload or video end. Resumes from last watched point.
- **Custom Completion Rule:** Activity completes only on full video watch (integrated with Moodle's completion system).
- **Attempts Page:** A new tab in the activity navbar showing a table of all users' attempts, including user name, attempt number, time watched, completion status, and timestamp.
- **Navbar Enhancement:** Adds "Page" (main content) and "View Attempts" tabs.

## Installation Steps

### Step 1: Implement Video Restrictions, Attempt Logging, and Database Table

1. **Create JavaScript File for Video Handling:**
   Create `mod/page/js/customvideo.js` with the following content:
   ```javascript
   function initCustomVideo({cmid, restrict}) {
       document.addEventListener('DOMContentLoaded', () => {
           const video = document.querySelector('video');
           if (!video) {
               console.error('Video not found');
               return;
           }

           let lastWatchedTime = 0;
           let hasLoggedAttempt = false;
           let maxWatched = 0;

           // Resume from last point
           fetch(`log_attempt.php?cmid=${cmid}`)
               .then(res => res.json())
               .then(data => {
                   if (data.status === 'success' && data.time_watched > 0) {
                       video.addEventListener('canplay', () => {
                           video.currentTime = data.time_watched;
                           maxWatched = data.time_watched;
                       }, { once: true });
                   }
               });

           // Log attempts
           const logAttempt = (time, complete) => {
               if (!hasLoggedAttempt) {
                   fetch('log_attempt.php', {
                       method: 'POST',
                       headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                       body: `cmid=${cmid}&time_watched=${time}&completed=${complete}`
                   });
                   hasLoggedAttempt = true;
               }
           };

           video.addEventListener('timeupdate', () => {
               if (!video.seeking) {
                   const current = video.currentTime;
                   lastWatchedTime = Math.max(lastWatchedTime, current);
                   maxWatched = Math.max(maxWatched, current);
               }
           });

           if (restrict) {
               video.addEventListener('seeking', () => {
                   if (video.currentTime > maxWatched + 0.01) {
                       video.currentTime = maxWatched;
                   }
               });
           }

           video.addEventListener('ended', () => {
               logAttempt(video.duration.toFixed(2), 1);
           });

           window.addEventListener('beforeunload', () => {
               if (lastWatchedTime > 0 && !video.ended) {
                   logAttempt(lastWatchedTime.toFixed(2), 0);
               }
           });

           video.addEventListener('play', () => {
               hasLoggedAttempt = false;
           });
       });
   }
   ```

2. **Edit Form for Restriction Setting:**
   In `mod/page/mod_form.php`, add in the definition function:
   ```php
   $mform->addElement('advcheckbox', 'restrictforwardseek', get_string('restrictforwardseek', 'page'));
   $mform->addHelpButton('restrictforwardseek', 'restrictforwardseek', 'page');
   $mform->setDefault('restrictforwardseek', 0);
   ```

3. **Edit View Script to Load JS:**
   In `mod/page/view.php`, before `echo $OUTPUT->header();`, add:
   ```php
   $restrictforward = !empty($page->restrictforwardseek);
   $PAGE->requires->js('/mod/page/js/customvideo.js');
   $PAGE->requires->js_init_code("initCustomVideo({cmid: {$cm->id}, restrict: " . ($restrictforward ? 'true' : 'false') . "});");
   ```

4. **Create Database Table for Attempts:**
   Run this SQL:
   ```sql
   CREATE TABLE mdl_page_attempts ( 
       id BIGINT(20) NOT NULL AUTO_INCREMENT, 
       pageid BIGINT(20) NOT NULL,  
       userid BIGINT(20) NOT NULL,  
       time_watched DECIMAL(10,2) NOT NULL, 
       completed TINYINT(4) NOT NULL DEFAULT 0,  
       timemodified BIGINT(20) NOT NULL,  
       PRIMARY KEY (id)  
   ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
   ```

5. **Create Attempt Logging Script:**
   Create `mod/page/log_attempt.php`:
   ```php
   <?php
   require_once(__DIR__ . '/../../config.php');
   require_once($CFG->libdir . '/completionlib.php');

   require_login();

   if ($_SERVER['REQUEST_METHOD'] === 'GET') {
       $cmid = required_param('cmid', PARAM_INT);

       $cm = get_coursemodule_from_id('page', $cmid, 0, false, MUST_EXIST);
       $page = $DB->get_record('page', ['id' => $cm->instance], '*', MUST_EXIST);

       $attempt = $DB->get_record_sql("
           SELECT time_watched 
           FROM {page_attempts} 
           WHERE pageid = ? AND userid = ? AND completed = 0 
           ORDER BY timemodified DESC LIMIT 1
       ", [$page->id, $USER->id]);

       if ($attempt) {
           echo json_encode(['status' => 'success', 'time_watched' => (float)$attempt->time_watched]);
       } else {
           echo json_encode(['status' => 'notfound', 'time_watched' => 0]);
       }
       exit;
   }

   $cmid = required_param('cmid', PARAM_INT);
   $time_watched = required_param('time_watched', PARAM_FLOAT);
   $completed = required_param('completed', PARAM_INT);

   error_log("DEBUG: Incoming POST - " . json_encode($_POST));

   $cm = get_coursemodule_from_id('page', $cmid, 0, false, MUST_EXIST);
   $course = $DB->get_record('course', ['id' => $cm->course], '*', MUST_EXIST);
   $page = $DB->get_record('page', ['id' => $cm->instance], '*', MUST_EXIST);

   // Avoid duplicate completion
   if ($DB->record_exists('page_attempts', ['pageid' => $page->id, 'userid' => $USER->id, 'completed' => 1])) {
       //error_log("DEBUG: User $USER->id already completed this page");
       echo json_encode(['status' => 'skipped', 'message' => 'User already has a completed attempt']);
       exit;
   }

   // Insert new attempt
   $record = new stdClass();
   $record->pageid = $page->id;
   $record->userid = $USER->id;
   $record->time_watched = $time_watched;
   $record->completed = $completed;
   $record->timemodified = time();

   $DB->insert_record('page_attempts', $record);
   error_log("DEBUG: Inserted new attempt for user $USER->id with completed = $completed");

   if ((int)$completed === 1) {
       error_log("DEBUG: Entering completion logic for user $USER->id");

       $modinfo = get_fast_modinfo($course, $USER->id);
       $cm_info = $modinfo->get_cm($cmid);
       $completion = new completion_info($course);

       $status = $completion->update_state($cm_info, true, $USER->id);
       error_log("DEBUG: Completion update result = " . var_export($status, true));

       $details = new \core_completion\cm_completion_details($completion, $cm_info, $USER->id);
       error_log("DEBUG: Completion details = " . json_encode($details->get_details()));
   }
   ?>
   ```

6. **Create Attempts View Page:**
   Create `mod/page/attempts.php`:
   ```php
   <?php
   require_once(__DIR__ . '/../../config.php');
   require_once($CFG->dirroot . '/mod/page/lib.php');

   $id = required_param('id', PARAM_INT);
   $cm = get_coursemodule_from_id('page', $id, 0, false, MUST_EXIST);
   $course = $DB->get_record('course', ['id' => $cm->course], '*', MUST_EXIST);
   $page = $DB->get_record('page', ['id' => $cm->instance], '*', MUST_EXIST);

   require_login($course, true, $cm);
   $context = context_module::instance($cm->id);
   //require_capability('mod/page:viewattempts', $context);

   $PAGE->set_url('/mod/page/attempts.php', ['id' => $cm->id]);
   $PAGE->set_title($page->name);
   $PAGE->set_heading($course->fullname);

   $attempts = $DB->get_records('page_attempts', ['pageid' => $page->id], 'userid ASC, timemodified ASC');

   echo $OUTPUT->header();
   echo $OUTPUT->heading(get_string('viewattempts', 'mod_page'));

   $table = new html_table();
   $table->head = ['User', 'Attempt', 'Time Watched', 'Completed', 'Date'];
   $table->data = [];

   $user_attempt_counts = [];
   $last_user_id = null;

   foreach ($attempts as $attempt) {
       $user = $DB->get_record('user', ['id' => $attempt->userid], 'id, firstname, lastname');

       if (!isset($user_attempt_counts[$user->id])) {
           $user_attempt_counts[$user->id] = 1;

           if ($last_user_id !== null) {
               $table->data[] = [
                   html_writer::tag('div', '', ['style' => 'border-top: 1px solid #ddd; margin: 4px 0; height: 1px']),
                   '', '', '', ''
               ];
           }

           $last_user_id = $user->id;
       } else {
           $user_attempt_counts[$user->id]++;
       }

       $profileurl = new moodle_url('/user/view.php', ['id' => $user->id, 'course' => $course->id]);
       $userlink = html_writer::link($profileurl, fullname($user));

       $table->data[] = [
           $userlink,
           $user_attempt_counts[$user->id],
           number_format($attempt->time_watched, 2),
           $attempt->completed ? 'Yes' : 'No',
           userdate($attempt->timemodified)
       ];
   }

   echo html_writer::table($table);
   echo $OUTPUT->footer();
   ?>
   ```

### Step 2: Add Custom Completion Rule for Video Completion

1. **Add Database Column:**
   Run this SQL:
   ```sql
   ALTER TABLE mdl_page  
   ADD COLUMN completionvideocompleted BOOLEAN NOT NULL DEFAULT FALSE;
   ```

2. **Edit Form for Completion Rule:**
   In `mod/page/mod_form.php`, add these methods:
   ```php
   /**
    * Add elements for setting the custom completion rules.
    *
    * @return array List of added element names or group names
    */
   public function add_completion_rules() {
       $mform = $this->_form;

       $group = [
           $mform->createElement(
               'checkbox',
               'completionvideocompletedenabled',
               '',
               get_string('completionvideo', 'page')
           )
       ];
       $mform->addGroup(
           $group,
           'completionvideocompletedgroup',
           get_string('completionvideogroup', 'page'),
           [' '],
           false
       );
       $mform->addHelpButton(
           'completionvideocompletedgroup',
           'completionvideo',
           'page'
       );

       return ['completionvideocompletedgroup'];
   }

   public function completion_rule_enabled($data) {
       $enabled = !empty($data['completionvideocompletedenabled']);
       return $enabled;
   }

   public function get_data() {
       $data = parent::get_data();
       if (!$data) {
           return $data;
       }
       $data->completionvideocompleted = !empty($data->completionvideocompletedenabled) ? 1 : 0;
       print_r($data);
       return $data;
   }
   ```

   In the existing `data_preprocessing` method, add:
   ```php
   parent::data_preprocessing($defaultvalues);
   if (isset($defaultvalues['completionvideocompleted'])) {
       $defaultvalues['completionvideocompletedenabled'] = (int)$defaultvalues['completionvideocompleted'];
   } 
   $defaultvalues['completion'] = COMPLETION_TRACKING_AUTOMATIC;
   ```

3. **Edit Library Functions:**
   In `mod/page/lib.php`, add to `page_supports`:
   ```php
   case FEATURE_COMPLETION_HAS_RULES:    return true;
   ```

   In `page_get_coursemodule_info`, add:
   ```php
   if ($coursemodule->completion == COMPLETION_TRACKING_AUTOMATIC) {
       if (!isset($info->customdata)) {
           $info->customdata = [];
       }
       if (!isset($info->customdata['customcompletionrules'])) {
           $info->customdata['customcompletionrules'] = [];
       }
       $info->customdata['customcompletionrules']['video_completed'] = $page->completionvideocompleted;
   }
   ```

4. **Create Custom Completion Class:**
   Create directory `mod/page/classes/completion/` and file `custom_completion.php`:
   ```php
   <?php
   declare(strict_types=1);

   namespace mod_page\completion;

   use core_completion\activity_custom_completion;
   use core_completion\cm_completion_details;

   class custom_completion extends activity_custom_completion {

       public function get_state(string $rule): int {
           global $DB;
           error_log("DEBUG: Entering get_state for rule {$rule}, user {$this->userid}, cm {$this->cm->id}");

           $this->validate_rule($rule);
           $userid = $this->userid;
           $cm = $this->cm;
           if ($rule === 'video_completed') {
               $completed = $DB->record_exists('page_attempts', [
                   'pageid' => $this->cm->instance,
                   'userid' => $this->userid,
                   'completed' => 1
               ]);
               //error_log("DEBUG: Video completion check = " . ($completed ? 'true' : 'false'));
               return $completed ? COMPLETION_COMPLETE : COMPLETION_INCOMPLETE;
           }

           error_log("DEBUG: Unknown rule {$rule}, returning INCOMPLETE fallback");
           return COMPLETION_INCOMPLETE;
       }

       public static function get_defined_custom_rules(): array {
           return ['video_completed'];
       }

       public function get_custom_rule_descriptions(): array {
           return [
               'video_completed' => get_string('completionvideo', 'page')
           ];
       }

       public function get_sort_order(): array {
           //error_log("DEBUG: Fetching sort order");
           return [
               'completionview',     
               'completionusegrade',  
               'completionpassgrade', 
               'video_completed'    
           ];
       }
   }
   ```

### Step 3: Override Navbar for Attempts Page

1. **Create Custom Secondary Navigation:**
   Create directory `mod/page/classes/navigation/views/` and file `secondary.php`:
   ```php
   <?php

   namespace mod_page\navigation\views;

   use core\navigation\views\secondary as core_secondary;
   use settings_navigation;
   use navigation_node;
   use moodle_url;

   class secondary extends core_secondary {

       protected function get_default_module_mapping(): array {
           $basenodes = parent::get_default_module_mapping();

           // TYPE_CUSTOM allows you to place your own items in order.
           $basenodes[self::TYPE_CUSTOM] += [
               'pageattempts' => 10, // Your custom key and its order
           ];

           return $basenodes;
       }

       protected function load_module_navigation(settings_navigation $settingsnav, ?navigation_node $rootnode = null): void {
           $rootnode = $rootnode ?? $this;

           $mainnode = $settingsnav->find('modulesettings', self::TYPE_SETTING);
           $nodes = $this->get_default_module_mapping();

           if ($mainnode) {
               // Main "Page content" tab
               $viewurl = new moodle_url('/mod/page/view.php', ['id' => $this->page->cm->id]);
               $setactive = $viewurl->compare($this->page->url, URL_MATCH_BASE);
               $modulepage = $rootnode->add(
                   get_string('mainpage', 'mod_page'),
                   $viewurl,
                   navigation_node::TYPE_SETTING,
                   null,
                   'modulepage'
               );

               if ($setactive) {
                   $modulepage->make_active();
               }

               $attempturl = new moodle_url('/mod/page/attempts.php', ['id' => $this->page->cm->id]);
               $attemptnode = $rootnode->add(
                   get_string('viewattempts', 'mod_page'), 
                   $attempturl,
                   navigation_node::TYPE_CUSTOM,
                   null,
                   'pageattempts',
                   new \pix_icon('i/report', '')
               );

               if ($attempturl->compare($this->page->url, URL_MATCH_BASE)) {
                   $attemptnode->make_active();
               }

               $orderednodes = $this->get_leaf_nodes($mainnode, $nodes);
               $this->add_ordered_nodes($orderednodes, $rootnode);
               $this->load_remaining_nodes($mainnode, $nodes, $rootnode);
           }
       }
   }
   ```

### Step 4: Add Language Strings
Add these to `mod/page/lang/en/page.php` (or your language file):
```php
$string['video_completed'] = 'Require video completion';
$string['video_completed_desc'] = 'The user must watch the entire video to complete this activity.';
$string['viewattempts'] = 'View Attempts';
$string['video_completed'] = 'Video Completion';
$string['video_completed_help'] = 'If enabled, the activity will be marked as complete when the learner finishes watching the video.';
$string['completionvideo'] = 'Require video completion';
$string['completionvideogroup'] = 'Video completion';
$string['completionvideo_help'] = 'The activity is marked complete when the user watches the video to completion.';
$string['viewattempts'] = 'View attempts';
$string['attempts'] = 'Attempts';
$string['mainpage'] = 'Page';
```

## Usage
1. **Create/Edit a Page Activity:**
   - Add a video to the page content.
   - Enable "Restrict forward seek" in activity settings to prevent skipping.
   - Under Completion Tracking, enable "Video completion" for custom rule.

2. **User Interaction:**
   - Users watch the video; progress is saved on unload or completion.
   - Completion marks only on full watch.

3. **View Attempts:**
   - In the activity, navigate to the "View Attempts" tab to see a table of all attempts.

## Debugging
- Check PHP error logs for `error_log` statements in scripts.
- Ensure JS console for video-related errors.
- Verify database inserts in `mdl_page_attempts`.

## Limitations
- Works only for videos embedded in Page activities.
- No capability checks on attempts page (uncomment `require_capability` if needed).
- Debug code (e.g., commented echoes) can be removed for production.

For issues, refer to Moodle docs on custom completion and navigation overrides.