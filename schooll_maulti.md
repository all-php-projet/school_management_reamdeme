# School Management System - Marks & Students Module

2. Helpers/helper.php
``
#install
composer require intervention/image:^2.7

# config/app.php
'providers' => [
    ...
    Intervention\Image\ImageServiceProvider::class,
],

'aliases' => [
    ...
    'Image' => Intervention\Image\Facades\Image::class,
],

``
```php
if (!function_exists('cropCompressImage')) {
    /**
     * Crop, compress, and convert an image to PNG
     *
     * @param string $file Uploaded file path
     * @param string $destination Path to save processed image
     * @param int $width Target width (default 300)
     * @param int $height Target height (default 300)
     * @param int $maxSizeKB Max size in KB (default 50)
     * @return string|null
     */
    function cropCompressImage($file, $destination, $width = 300, $height = 300, $maxSizeKB = 50)
    {
        $info = getimagesize($file);
        if (!$info) return null;

        $mime = $info['mime'];
        $img = null;

        switch ($mime) {
            case 'image/jpeg':
                $img = imagecreatefromjpeg($file);
                break;
            case 'image/png':
                $img = imagecreatefrompng($file);
                break;
            case 'image/gif':
                $img = imagecreatefromgif($file);
                break;
            case 'image/webp':
                if (function_exists('imagecreatefromwebp')) {
                    $img = imagecreatefromwebp($file);
                } else {
                    return null;
                }
                break;
            default:
                return null;
        }

        $srcWidth = imagesx($img);
        $srcHeight = imagesy($img);

        // Create a square crop
        $minDim = min($srcWidth, $srcHeight);
        $cropX = ($srcWidth - $minDim) / 2;
        $cropY = ($srcHeight - $minDim) / 2;

        $cropImg = imagecreatetruecolor($width, $height);

        // Preserve transparency
        imagealphablending($cropImg, false);
        imagesavealpha($cropImg, true);
        $transparent = imagecolorallocatealpha($cropImg, 0, 0, 0, 127);
        imagefilledrectangle($cropImg, 0, 0, $width, $height, $transparent);

        // Crop & resize
        imagecopyresampled($cropImg, $img, 0, 0, $cropX, $cropY, $width, $height, $minDim, $minDim);

        // Compress PNG
        $quality = 9; // PNG compression level 0-9 (higher is smaller)
        do {
            ob_start();
            imagepng($cropImg, null, $quality);
            $imgData = ob_get_clean();
            $sizeKB = strlen($imgData) / 1024;
            $quality = max(0, $quality - 1);
        } while ($sizeKB > $maxSizeKB && $quality > 0);

        file_put_contents($destination, $imgData);
        imagedestroy($img);
        imagedestroy($cropImg);

        return $destination;
    }
}

```

## Multi mark

# multiCreate.blade.php
# marksheet.blade.php
```php
   Route::get('marks/multi/create', [MarkController::class, 'multiCreate'])->name('mark.multiCreate');
   Route::get('marks/multi/marksheet', [MarkController::class, 'marksheet'])->name('mark.marksheet');
   
    public function multiCreate()
    {
        $exams = Exam::all();
        $classes = StudentClass::all();
        $subjects = Subject::all();
        $students = Student::all();

        return view('mark.multiCreate', compact('exams', 'classes', 'subjects', 'students'));
    }

    public function marksheet(Request $request)
    {
        $class_id = $request->class_id;
        $students = Student::where('class_id', $class_id)->get();

        $subjects = Subject::where('class_id', $class_id)
            ->get();

        $fourthSubjects = FourthSubject::where('class_id', $class_id)
            ->where('is_active', true)
            ->get();
        $fourth_subjects = [];
        if ($fourthSubjects->isNotEmpty()) {
            $fourth_subjects = Subject::where('class_id', $class_id)->where('deleted', false)->where('compulsory', false)->get()->toArray();
        }
        $studentSubjects = StudentSubject::whereIn('student_id', $students->pluck('id'))
            ->get()
            ->groupBy('student_id');
        $exam_id = $request->exam_id;


        return response()->json([
            'html' => view('mark.marksheet', compact(
                'students',
                'subjects',
                'fourthSubjects',
                'fourth_subjects',
                'studentSubjects',
                'exam_id'
            ))->render()
        ]);
    }

    Route::post('marks/multi/storeMarks', [MarkController::class, 'storeMarks'])->name('mark.storeMarks');
    public function storeMarks(Request $request)
    {
        $data = $request->input('students', []);
        foreach ($data as $studentId => $studentData) {
            $classId = $studentData['class_id'];
            $roll = $studentData['recent_roll'];
            $section_id = $request->section_id;
            foreach ($studentData['marks'] ?? [] as $subjectId => $subjectMarks) {
                foreach ($subjectMarks as $type => $marks) {
                    // Calculate Achieved Mark
                    $achived = collect($marks)->sum();

                    Mark::updateOrCreate(
                        [
                            'student_id' => $studentId,
                            'subject_id' => $subjectId,
                            'exam_id'    => $request->exam_id,
                        ],
                        [
                            'class_id'     => $classId,
                            'section_id'   => $section_id,
                            'recent_roll'  => $roll,
                            'writen'       => $marks['writen'] ?? 0,
                            'mcq'          => $marks['mcq'] ?? 0,
                            'practical'    => $marks['practical'] ?? 0,
                            'assignment'   => $marks['assignment'] ?? 0,
                            'attendance'   => $marks['attendance'] ?? 0,
                            'class_test'   => $marks['class_test'] ?? 0,
                            'achived_mark' => $achived,
                        ]
                    );
                }
            }
        }

        return response()->json(['message' => 'Marks saved successfully.']);
    }

```





## Subject
# multi_create.blade.php
# student_subject_rows.blade.php
```php
    # multi_create.blade.php
    # student_subject_rows.blade.php

    Route::post('student/subjects/students', [StudentSubjectController::class, 'multi_student_subject'])->name('student.subject.multi_student_subject');
    Route::get('student/subjects/students', [StudentSubjectController::class, 'getStudentSubject'])->name('student.subject.getStudentSubject');
    Route::get('student/subjects/multi_create', [StudentSubjectController::class, 'multiCreate'])->name('student.subject.multiCreate');

   public function multiCreate()
    {
        $classes = StudentClass::get();
        $sections = Section::get();
        return view('studentSubject.multi_create', compact('classes', 'sections'));
    }

    public function getStudentSubject(Request $request)
    {
        $class_id = $request->class_id;
        $students = Student::where('class_id', $class_id)->get();

        $subjects = Subject::where('class_id', $class_id)
            ->get();

        $fourthSubjects = FourthSubject::where('class_id', $class_id)
            ->where('is_active', true)
            ->get();
        $fourth_subjects = [];
        if ($fourthSubjects->isNotEmpty()) {
            $fourth_subjects = Subject::where('class_id', $class_id)->where('deleted', false)->where('compulsory', false)->get()->toArray();
        }
        $studentSubjects = StudentSubject::whereIn('student_id', $students->pluck('id'))
            ->get()
            ->groupBy('student_id');

        return response()->json([
            'html' => view('studentSubject.student_subject_rows', compact(
                'students',
                'subjects',
                'fourthSubjects',
                'fourth_subjects',
                'studentSubjects'
            ))->render()
        ]);
    }
     public function multi_student_subject(Request $request)
    {
        $studentsData = $request->input('students', []);

        foreach ($studentsData as $data) {
            $studentId = $data['student_id'];

            // Get selected subject IDs (normal subjects)
            $subjectIds = $data['subject_ids'] ?? [];

            // 🔥 Delete old subjects that are NOT in the current list
            StudentSubject::where('student_id', $studentId)
                ->whereNotIn('subject_id', $subjectIds)
                ->delete();

            // 🔥 Save all selected subjects
            foreach ($subjectIds as $subjectId) {
                StudentSubject::updateOrCreate(
                    [
                        'student_id' => $studentId,
                        'subject_id' => $subjectId
                    ],
                    [
                        'recent_roll' => $data['recent_roll'],
                        'class_id' => $data['class_id'],
                        'section_id' => $request->section_id ?? null,
                    ]
                );
            }

            // 🔥 Handle fourth subject
            if (!empty($data['fourth_subject_id'])) {
                StudentSubject::updateOrCreate(
                    [
                        'student_id' => $studentId,
                        'fourth_subject_id' => $data['fourth_subject_id']
                    ],
                    [
                        'recent_roll' => $data['recent_roll'],
                        'class_id' => $data['class_id'],
                        'section_id' => $request->section_id ?? null,
                    ]
                );
            } else {
                // 🔥 Remove fourth subject if not selected anymore
                StudentSubject::where('student_id', $studentId)
                    ->whereNotNull('fourth_subject_id')
                    ->delete();
            }
        }

        return response()->json(['status' => 'success', 'message' => 'Student subjects saved!']);
    }
```
## Student Create
# multi-student.blade.php
# create_multi_form.blade.php
```php
    # route
    Route::get('students/create_multi', [StudentController::class, 'create_multi'])->name('students.create_multi');
    Route::post('students/store_multi', [StudentController::class, 'store_multi'])->name('students.store_multi');
    Route::get('students/create_multi_form', [StudentController::class, 'create_multi_form'])->name('students.create_multi_form');

    ##Controller
     public function create_multi()
    {
        $genders = Gender::all();
        $groups = StudentGroup::all();
        $classes = StudentClass::all();
        $sections = Section::all();
        $religions = Religion::all();
        return view('student.multi-student', compact('genders', 'classes', 'groups', 'sections', 'religions'));
    }
    public function create_multi_form(Request $request)
    {
        $genders = Gender::all();
        $groups = StudentGroup::all();
        $classes = StudentClass::all();
        $sections = Section::all();
        $religions = Religion::all();
        $SI_num = $request->SI_num;
        return response()->json([
            'html' => view('student.create_multi_form', compact('genders', 'classes', 'groups', 'sections', 'religions', 'SI_num'))->render()
        ]);
    }
    public function store_multi(Request $request)
    {
        // 🔹 Loop through each student and save
        foreach ($request->name as $index => $name) {
            $imagePath = null;
            if ($request->hasFile("picture.$index")) {
                $file = $request->file("picture.$index");
                if ($file->isValid()) {
                    $filename = time() . "_$index.png";
                    $destination = public_path('uploads/student/' . $filename);

                    cropCompressImage($file->getPathname(), $destination, 300, 300, 50);

                    $imagePath = url('uploads/student/' . $filename);
                }
            }
            // Save Student
            $student = Student::create([
                'uniqe_id' => $request->uniqe_id[$index] ?? null,
                'roll' => $request->roll[$index] ?? null,
                'name' => $name,
                'picture' => $imagePath,
                'father_name' => $request->father_name[$index],
                'mother_name' => $request->mother_name[$index],
                'phone_number' => $request->phone_number[$index],
                'date_of_birth' => $request->date_of_birth[$index],
                'address' => $request->address[$index],
                'blood_group' => $request->blood_group[$index] ?? null,
                'class_id' => $request->class_id[$index],
                'section_id' => $request->section_id[$index] ?? null,
                'student_group_id' => $request->student_group_id[$index] ?? null,
                'gender_id' => $request->gender_id[$index],
                'religion_id' => $request->religion_id[$index],
                'initial_due' => $request->initial_due[$index] ?? 0,
                'present_due' => $request->present_due[$index] ?? 0,
                'current_admission_date' => $request->current_admission_date[$index],
                'account_active_date' => $request->account_active_date[$index],
            ]);
            // Save Student Record
            StudentRecord::create([
                'student_id' => $student->id,
                'class_id' => $request->class_id[$index],
                'section_id' => $request->section_id[$index] ?? null,
                'academic_year' => date('Y'),
                'class_roll' => $request->roll[$index] ?? null,
                'account_active_date' => $request->account_active_date[$index],
            ]);
        }
        return response()->json(['success' => 'All students saved successfully.']);
    }
```

<!-- sidebar  -->
## Search this file
# vendor\crocodicstudio\crudbooster\src\views\sidebar.blade.php
1. student.subject.multiCreate
2. students.create_multi
3. student.subject.multiCreate
