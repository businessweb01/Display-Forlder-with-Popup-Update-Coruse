# Display-Forlder-with-Popup-Update-Coruse
<?php
session_start();
include '../conn.php';

// Retrieve instructor's name from the database
$email = $_SESSION['email'] ?? 'default_user';
$retreive_name = "SELECT instructor_name FROM instructor WHERE email = ?";
$stmt = $conn->prepare($retreive_name);
$stmt->bind_param("s", $email);
$stmt->execute();
$result = $stmt->get_result();
if ($result->num_rows > 0) {
    $name = $result->fetch_assoc()['instructor_name'];
    $_SESSION['name_instructor'] = $name;
} else {
    echo "Error: Instructor not found.";
    exit;
}
$stmt->close();

// Define the path to the user's folder
$userFolderPath = __DIR__ . "/../courses/$email";

// Fetch course details from the database
$sql = "SELECT courseName, difficulty, description, lessons, course_ratings, thumbnail, courseId FROM courses WHERE instructor_email = ?";
$stmt = $conn->prepare($sql);
$stmt->bind_param("s", $email);
$stmt->execute();
$stmt->bind_result($coursename, $difficulty, $description, $lessons, $course_ratings, $thumbnail, $courseId);

// Prepare an associative array to hold course details
$courses = [];
while ($stmt->fetch()) {
    $courses[$coursename] = [
        'courseName' => $coursename,
        'description' => $description,
        'lessons' => $lessons,
        'difficulty' => $difficulty,
        'course_ratings' => $course_ratings,
        'thumbnail' => $thumbnail,
        'courseId' => $courseId
    ];
}
$stmt->close();

// Define the base URL path to the thumbnails
$thumbnailBaseUrl = "/UploadCourses/Course-Thumbnails";

// Check if the directory exists
if (is_dir($userFolderPath)) {
    $dir = opendir($userFolderPath);
    while (($file = readdir($dir)) !== false) {
        if ($file != '.' && $file != '..') {
            $courseDetails = $courses[$file] ?? [
                'description' => 'No description available',
                'lessons' => '0',
                'difficulty' => 'Unknown',
                'course_ratings' => 0,
                'thumbnail' => '',
                'courseId' => '',
                'courseName' => $file
            ];

            $isSelected = ($file === ($_SESSION['selected_course'] ?? '')) ? 'selected' : '';
            $fullStars = floor($courseDetails['course_ratings']);
            $halfStar = ceil($courseDetails['course_ratings']) - $fullStars;
            $thumbnailPath = "$thumbnailBaseUrl/$email/$file/{$courseDetails['thumbnail']}";
            if (!file_exists(__DIR__ . "/../Course-Thumbnails/$email/$file/{$courseDetails['thumbnail']}")) {
                $thumbnailPath = '../default_thumbnail.png';
            }
            $isMarquee = '';
            $textLength = strlen($file);
            $maxLength = 25;
            if ($textLength > $maxLength) {
                $isMarquee = 'marquee';
            }

            echo "<div class='course-container'>";
            echo "<img data-src='" . htmlspecialchars($thumbnailPath, ENT_QUOTES, 'UTF-8') . "' alt='Course Thumbnail' class='course-thumbnail lazyload'>";
            echo "<div class='course-name-wrapper $isMarquee'>";
            echo "<h3 class='course-name'>" . htmlspecialchars($file, ENT_QUOTES, 'UTF-8') . "</h3>";
            echo "</div>";
            echo "<div class='starandtitle'>";
            echo "<div class='course-star-rating'>";
            for ($i = 1; $i <= 5; $i++) {
                $starType = 'empty';
                if ($i <= $fullStars) {
                    $starType = 'full';
                } elseif ($i == $fullStars + 1 && $halfStar > 0) {
                    $starType = 'half';
                }
                $starSvg = '<svg width="1.25rem" height="1.25rem" viewBox="0 0 24 24" fill="#ffb633"> <path d="M12 2.3l2.4 7.4h7.6l-6 4.8 2.3 7.4-6.3-4.7-6.3 4.7 2.3-7.4-6-4.8h7.6z"/> </svg>';
                echo "<input type='radio' id='star$i' name='rating' value='$i' class='star-input' style='display: none;'>";
                echo "<label for='star$i' class='star-label $starType'>$starSvg</label>";
            }
            echo "</div>";
            echo "<p>Ratings " . htmlspecialchars($courseDetails['course_ratings'], ENT_QUOTES, 'UTF-8') . "/5" . "</p>";
            echo "</div>";
            echo "<form method='post' action='actions/view_folder.php?coursename=" . htmlspecialchars($file, ENT_QUOTES, 'UTF-8') . "'>";
            echo "<button class='view-details' id='view-details'>View Course</button>";
            echo "<input type='hidden' name='coursename' value='" . htmlspecialchars($file, ENT_QUOTES, 'UTF-8') . "'>";
            echo "<input type='hidden' name='email' value='" . htmlspecialchars($email, ENT_QUOTES, 'UTF-8') . "'>";
            echo "</form>";
            echo "<div class='course-buttons'>";
            echo "<form method='post' action='' id='deleteForm' class='deleteForm'>";
            echo "<input type='hidden' name='coursename' value='" . htmlspecialchars($file, ENT_QUOTES, 'UTF-8') . "'>";
            echo "<input type='hidden' name='email' value='" . htmlspecialchars($email, ENT_QUOTES, 'UTF-8') . "'>";
            echo "<button type='submit' name='deletefolder' class='delete-folder'>Delete</button>";
            echo "</form>";
            echo "<button class='updateCourse-Button' onclick=\"openUpdateForm('" . htmlspecialchars($file, ENT_QUOTES, 'UTF-8') . "', '" . htmlspecialchars($courseDetails['courseId'], ENT_QUOTES, 'UTF-8') . "', '" . htmlspecialchars($courseDetails['description'], ENT_QUOTES, 'UTF-8') . "', '" . htmlspecialchars($courseDetails['difficulty'], ENT_QUOTES, 'UTF-8') . "', '" . htmlspecialchars($courseDetails['lessons'], ENT_QUOTES, 'UTF-8') . "')\">Update</button>";
            echo "</div>";
            echo "</div>";
            echo '<div id="overlay-update-course" class="overlay-update-course" style="display: none;">
                    <div class="updateCourse popup-update-form">
                        <div class="closeForm">
                            <box-icon name="x" class="cancelCourseSubmit" size="2rem" onclick="closeUpdateForm()"></box-icon>
                        </div>
                        <form action="" method="post" enctype="multipart/form-data" id="update-course-form">
                            <input type="hidden" name="email" value="' . htmlspecialchars($email, ENT_QUOTES, 'UTF-8') . '">
                            <input type="hidden" name="courseId" id="courseId" value="' . htmlspecialchars($courseDetails['courseId'], ENT_QUOTES, 'UTF-8') . '">
                            <label for="course-thumbnail">Course Thumbnail:</label>
                            <input type="file" name="course-thumbnail" id="course-thumbnail-input">
                            <input type="hidden" name="course-name" id="course-name-input" value="' . htmlspecialchars($file, ENT_QUOTES, 'UTF-8') . '">
                            <label for="new-course-name">Course Name:</label>
                            <input type="text" name="new-course-name" id="new-course-name-input" value="' . htmlspecialchars($courseDetails['courseName'], ENT_QUOTES, 'UTF-8') . '">
                            <label for="CourseDifficulty">Course Difficulty:</label>
                            <select name="CourseDifficulty" id="CourseDifficulty-input">
                                <option value="Beginner" ' . ($courseDetails['difficulty'] == 'Beginner' ? 'selected' : '') . '>Beginner</option>
                                <option value="Intermediate" ' . ($courseDetails['difficulty'] == 'Intermediate' ? 'selected' : '') . '>Intermediate</option>
                                <option value="Expert" ' . ($courseDetails['difficulty'] == 'Expert' ? 'selected' : '') . '>Expert</option>
                            </select>
                            <label for="numberoflessons">Number of Lessons:</label>
                            <input type="number" name="numberoflessons" id="numberoflessons-input" value="' . htmlspecialchars($courseDetails['lessons'], ENT_QUOTES, 'UTF-8') . '">
                            <label for="course-description">Course Description:</label>
                            <textarea name="course-description" id="course-description-input">' . htmlspecialchars($courseDetails['description'], ENT_QUOTES, 'UTF-8') . '</textarea>
                            <div class="updateCourseButtons">
                                <button type="submit" id="updateCourseSubmit" class="updateCourseSubmit">Update</button>
                            </div>
                        </form>
                    </div>
                </div>';
        }
    }
    closedir($dir);
} else {
    echo "Directory does not exist.";
}

$conn->close();
?>

<style>
    .course-name-wrapper {
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    position: relative;
    max-width: 100%;
    cursor: pointer;
}

.course-name-wrapper .course-name {
    white-space: pre;
}

.course-name-wrapper.marquee:hover .course-name {
    animation: marquee 5s linear infinite;
}

@keyframes marquee {
    0% { transform: translateX(0%); }
    100% { transform: translateX(-100%); }
}
.lazyload {
    opacity: 0;
    transition: opacity 0.5s ease-in-out;
}

.lazyload.lazyloaded {
    opacity: 1;
}

/* CSS for fade-in animation */

</style>
<script src="node_modules/jquery/dist/jquery.min.js"></script>
<script src="node_modules/lazysizes/lazysizes.min.js"></script>
<script src="node_modules/sweetalert2/dist/sweetalert2.all.min.js"></script>

<script>
 $(document).ready(function() {
    // Handle form submission for updating course details
    $('#update-course-form').on('submit', function(e) {
        e.preventDefault(); // Prevent the default form submission
        var formData = new FormData(this);
        $.ajax({
            url: 'actions/update_course.php', // Path to your PHP script
            type: 'POST',
            data: formData,
            processData: false,
            contentType: false,
            dataType: 'json', // Expect JSON response
            success: function(response) {
                if (response.status === 'success') {
                    Swal.fire({
                        icon: 'success',
                        title: 'Success!',
                        text: response.message
                    }).then(() => {
                        resetForm();
                        closeUpdateForm();
                        loadCourses(); // Update the course list
                        fetchLatestImage(); // Fetch the latest image URL
                    });
                } else {
                    Swal.fire({
                        icon: 'error',
                        title: 'Oops...',
                        text: response.message
                    });
                }
            },
            error: function(xhr, status, error) {
                console.error('AJAX request failed:', xhr, status, error);
                Swal.fire({
                    icon: 'error',
                    title: 'Error!',
                    text: 'An unexpected error occurred.'
                });
            }
        });
    });

    // Function to open the update form
    window.openUpdateForm = function(courseName, courseId, description, difficulty, lessons) {
        $('#courseId').val(courseId);
        $('#course-name-input').val(courseName);
        $('#new-course-name-input').val(courseName);
        $('#CourseDifficulty-input').val(difficulty);
        $('#numberoflessons-input').val(lessons);
        $('#course-description-input').val(description);
        $('#overlay-update-course').show();
    };

    // Function to close the update form
    window.closeUpdateForm = function() {
        $('#overlay-update-course').hide();
    };

    // Function to fetch and update the latest thumbnail
    function fetchLatestImage() {
        $.ajax({
            url: 'actions/fetch_new_image.php',
            type: 'GET',
            dataType: 'json',
            success: function(response) {
                if (response.thumbnailUrl) {
                    $('#course-thumbnail').each(function() {
                        var currentSrc = $(this).attr('data-src');
                        if (currentSrc && currentSrc !== response.thumbnailUrl) {
                            $(this).attr('data-src', response.thumbnailUrl);
                            $(this).removeClass('lazyload'); // Trigger the lazyload refresh
                        }
                    });
                }
            },
            error: function(xhr, status, error) {
                console.error('Failed to fetch latest image:', xhr);
                console.error('Response Text:', xhr.responseText);
            }
        });
    }

    // Fetch the latest image every 2 seconds
    setInterval(fetchLatestImage, 2000);
    fetchLatestImage();

    // Reset the form after submission
    function resetForm() {
        $('#update-course-form')[0].reset();
    }
});
</script>
