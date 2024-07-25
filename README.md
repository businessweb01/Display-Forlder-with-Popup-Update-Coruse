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



// CSS
<style>
    *{
    margin: 0;
    padding: 0;
    box-sizing: border-box;
    font-family: 'Poppins', sans-serif;
}
body{
    background:linear-gradient(180deg, #C8C1AC,#FFE794);
    min-height: 100vh;
    width: 100%;
}
body::-webkit-scrollbar {
   width: 12px;
}

body::-webkit-scrollbar-track {
   background: #f1f1f100;
   border-radius: 10px;
}

body::-webkit-scrollbar-thumb {
   background: #3c3428;
   border-radius: 10px;
}
/* Navbar */
.header{
    width: 100%;
    height: fit-content;
}
.navigation{
    display: flex;
    background-color: #1a1a1a;
    padding: 20px 10px;
    justify-content: space-between;
    align-items: center;
}
.nav-links{
    display: flex;
    gap: 2rem;
    margin-inline: 2rem;
}
.nav-link{
    color: rgba(255, 255, 255, 0.7);
    text-decoration: none;
    font-size: clamp(0.7rem, 1vw, 1.5rem);
    transition: 0.5s ease;
}
.nav-link:hover{
    color:rgba(255, 255, 255, 1);
}
.nav-icons{
    display: flex;
    gap: 1rem;
    margin-right: 2rem;
}
.bell-icon:hover{
    animation: tada 1s;
}
@keyframes tada {
    0% {
        transform: scale(1);
    }
    10%, 20% {
        transform: scale(0.9) rotate(-3deg);
    }
    30%, 50%, 70%, 90% {
        transform: scale(1.1) rotate(3deg);
    }
    40%, 60%, 80% {
        transform: scale(1.1) rotate(-3deg);
    }
    100% {
        transform: scale(1) rotate(0);
    }
}
@media screen and (max-width: 768px) {
   .nav-links{
       margin-inline: 1rem;
       gap: 1rem;
   }
    
}
/* Style for the popup */
.popup {
    display: none;
    position: fixed;
    top: 0;
    right: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5);
    z-index: 1000;
}

.popup-content {
    position: absolute;
    top: 10px;
    right: 10px;
    background-color: white;
    padding: 20px;
    border-radius: 10px;
    box-shadow: 0 5px 15px rgba(0, 0, 0, 0.3);
    width: 300px;
    
}
.close {
    top:0;
    left:0;

    position: absolute;
    font-size: 24px;
    cursor: pointer;
    color: #000000;
}

.userpicname {
    display: flex;
    flex-direction: row;
    align-items: center;
    margin-bottom: 1rem;

}
.userpic {
    width: 50px;
    height: 50px;
    margin-top:1rem;
    background-image: url('dp.png');
    background-size: cover;
    background-position: center;
    border-radius: 50%;
    margin-bottom: 10px;
}

.username {
    margin-left: 1rem;
    font-size: 20px;
    font-weight: bold;
}

.divider {
    width: 100%;
    align-self: center;
    background-color: #7CBE28;
    border: none;
    border-radius: 1rem;
    height: 0.2rem;
}
.settings{
    margin-top: 1rem;
    margin-bottom: 1rem;
}

.IconLink{
    cursor: pointer;
    text-decoration: none;
    color: #000000;
}   
.editprofile{
    display: flex;
    flex-direction: row;
    align-items: center;
    margin-bottom: 1rem;
    gap: 0.5rem;
}
.lightdark {
    width: fit-content;
    border-radius: 50px;
    display: flex;
   gap: 0.5rem;
}
.lightdark p{
    margin-right: 0.2rem;

}
.light{
    padding: 5px 5px;
    display: flex;
    align-items: center;
    justify-content: center;
    height: fit-content;
    width: fit-content;
    border-radius: 50px;
    border:none;
    background: #ffffff;
    box-shadow: inset 5px 5px 10px #b3b3b3,
                inset -5px -5px 10px #ffffff;
}
.dark{
    padding: 5px 5px;
    display: flex;
    align-items: center;
    justify-content: center;
    height: fit-content;
    width: fit-content;
    border-radius: 50px;
    border:none;
    background: #ffffff;
    box-shadow: inset 5px 5px 10px #b3b3b3,
                inset -5px -5px 10px #ffffff;
}
.sun{
    border-radius: 50px;
    margin-right: 0.2rem;
}   
.moon{
    border-radius: 50px;
}
.exitdashboard{
    margin-top: 1rem;
    display: flex;
    flex-direction: row;
    align-items: center;
    margin-bottom: 1rem;
    gap: 0.5rem;
}

/* Popup Edit */
.closeandlogo{
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: space-between;
    width: 100%;
    margin-bottom: 1rem;
}
.Popuplogo span{
    font-size: clamp(1rem, 1.5vw, 1.75rem);
    font-weight: 600;
    font-style: italic;
}
.popupEdit{
    position: fixed;
    z-index: 1000;
    display: none;
    flex-direction: column;
    top: 20%;
    left: 50%;
    transform: translate(-50%, -50%);
    width: fit-content;
    height: fit-content;
    align-items: center;
}
.popupEdit.active {
    display: block;
}
.modal-overlay{
    display: none; /* Initially hidden */
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(175, 175, 175, 0); /* White with 80% opacity */
    backdrop-filter: blur(10px); /* Apply a blur effect */
    z-index: 999
}
.modal-overlay.active {
    display: block;
}
.edit-content{
    display: flex;
    flex-direction: column;
    margin-top: 5rem;
    width: 100%;
    max-width: 350px;
    height: fit-content;
    background:linear-gradient(180deg, #FFFFFF,#FEFFDF);    
    padding: 10px;
    border-radius: 10px;
    box-shadow: 3.3px 3.3px 1px #77787A, -3.3px -3.3px 1px #FFFFFF;
    margin-right: 5rem;
    margin-left: 5rem;
}
.inputs{
    display: flex;
}
.editForm input[type="text"],
.editForm input[type="email"],
.editForm input[type="tel"]{
    border: none;
    background: none;
    outline: none;
    width: 100%;
}
.editForm input[type="email"]{
    width: 200px;
}
.editForm input.editable {
    border-bottom: 1px solid black;
}
.userpicnameemail{
    display: flex;
    flex-direction: row;
    align-items: center;
    margin-bottom: 1rem;
}
.Edit-userpic {
    width: 50px;
    height: 50px;
    margin-top:1rem;
    margin-left: 1rem;
    background-image: url('dp.png');
    background-size: cover;
    background-position: center;
    border-radius: 50%;
    margin-bottom: 10px;
}
.usernameemail{
    display: flex;
    flex-direction: column;
    margin-left: 1rem;
    gap: 0.5rem;
}
.addressphone{
    width: 300px;
    display: flex;
    flex-direction: column;
    margin-left: 1rem;
    gap: 0.5rem;
}
.submit-edit{
    margin-top: 1rem;
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: center;
    margin-bottom: 1rem;
    gap: 0.5rem;
}
.submit-edit button{
    padding: 5px 10px;
    border-radius: 5px;
    border: none;
    background:linear-gradient(180deg, #17FF73,#D6EE43);
    color: #000000;
    font-weight: 600;
    box-shadow: 3.6px 3.6px 2px #B5B6B9, -3.6px -3.6px 2px #FFFFFF;

}

/* End of Popup */

/* End of Navbar */

.logo{
    /* margin-top: 1rem; */
    text-align: center;
    font-weight: 600;
    font-size: clamp(2rem, 2vw, 3rem);

}
.Agri{
    background: url(grass.jpg);
    background-size: contain;
    -webkit-text-fill-color: transparent;
    background-clip: text;
    
}
/* start of content */
.content {
    margin-top: 2rem;
    margin-inline: 2rem;
    display: flex;
}
/* Start of box1*/
.box1 {
    padding: 10px 10px;
    width: 300px;
    height: fit-content;
    border-radius: 10px;
    background-color:#efefef;
    box-shadow: 3.3px 3.3px 1px #77787A, -3.3px -3.3px 1px #FFFFFF;
    margin-right: 1rem;
}
.box1-buttons{
    margin-top: 2rem;
    display: flex;
    flex-direction: column;
    gap: 1rem;
    align-items: center;
}
.add-course{
    font-size: clamp(1rem, 1vw, 1.5rem);
    font-weight: 600;
    padding: 10px 15px;
    max-width: 200px;
    width: 100%;
    height: fit-content;
    border-radius: 10px;
    border: none;
    background-color: #dbdbdba3;
    box-shadow: 3.3px 3.3px 1px #77787A, -3.3px -3.3px 1px #FFFFFF;
    display: flex;
    align-items: center;
    gap: 0.5rem;
}
.archives{
    font-size: clamp(1rem, 1vw, 1.5rem);
    font-weight: 600;
    padding: 10px 10px;
    max-width: 200px;
    width: 100%;
    height: fit-content;
    border: none;
    border-radius: 10px;
    background-color: #dbdbdba3;
    box-shadow: 3.3px 3.3px 1px #77787A, -3.3px -3.3px 1px #FFFFFF;
    display: flex;
    align-items: center;
    gap: 0.5rem;
}
.storage-info {
    margin: 20px;
    padding: 10px;
    border-radius: 5px;
}

.progress-bar {
    height: 10px;
    background-color: #ddd;
    border-radius: 10px;
    overflow: hidden;
}

.progress {
    height: 100%;
    width: 0%;
    background-color: #2a2ccb; /* Green color */
    transition: width 0.3s ease-in-out; /* Smooth transition */
}

.storage-info p {
    margin: 10px 0;
    font-size: 16px;
}
.button-clicked{
    background-color: #ffdd79;
    box-shadow: inset 5px 5px 6px #c8ba64,
            inset -5px -5px 6px #FEDF91;
}
/* end of box1 */

/* start of box2 */
.box2 {
    padding: 10px 10px;
    width: 100%;
    height: 600px;
    border-radius: 10px;
    background-color: #efefef;
    box-shadow: 3.3px 3.3px 1px #77787A, -3.3px -3.3px 1px #FFFFFF;
    display: flex;
    flex-direction: column; /* Arrange items in a column */
    align-items: center; /* Center align all child elements horizontally */
}
.back{
    align-self: start;
    display: flex;
    align-items: center;
    width: fit-content;
}
.back a{
    text-decoration: none;
    color: black;
    display: flex;
    align-items: center;
}
.back p{
    font-size: clamp(1rem, 1vw, 1.5rem);
    font-weight: 600;
    margin-left: 0.5rem;
    opacity: 0; /* Initially transparent */
    transform: translateX(-15px); /* Position it to the right initially */
    transition: opacity 0.3s ease-in, transform 0.3s ease-in;
    cursor: pointer;
    color:rgba(0, 0, 0, 0.5);
}
.back:hover p{
    opacity: 1;
    transform: translateX(0);
}
.box2 h1{
    font-size: clamp(2rem, 2.5vw, 3rem);
    text-align: center;
}
.search-container {
    margin-right: auto;
    margin-top: 1rem;
    margin-left: 1rem;
    position: relative;
    display: inline-block;
    width: 100%;
    max-width: 350px;
}
.search-container button {
    position: absolute;
    top: 0;
    left: 0;
    height: 100%;
    width: 40px; /* Adjust width to match the padding-left of input */
    border: none;
    background: none;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
}

.search-container input[type="text"] {
    width: 100%;
    padding: 10px;
    padding-left: 45px; /* Adjust padding to make room for the icon */
    font-size: clamp(0.7rem, 1vw, 1.5rem);
    box-sizing: border-box; /* Ensures padding is included in width */
    border-radius: 2rem;
}
.search-container .search-icon {
    position: absolute;
    top: 50%;
    left: 10px;
    transform: translateY(-50%);
    font-size: clamp(0.7rem, 1vw, 1.5rem);    
    color: #888;
    pointer-events: none; /* Ensures the icon doesn't block input clicks */
}

@media (max-width: 700px) {
    .search-container {
        margin-top: 1rem;
        margin-left: 0.5rem;
        max-width: 200px;
    }
    .search-container input[type="text"] {
        padding: 8px;
        padding-left: 30px; /* Adjust padding to make room for the icon */
    }

    .search-container .search-icon {
        left: 8px;
    }
}

@media (max-width: 400px) {
    .search-container input[type="text"] {
        padding: 6px;
        padding-left: 25px; /* Adjust padding to make room for the icon */
        font-size: 12px;
    }

    .search-container .search-icon {
        left: 6px;
        font-size: 12px;
    }
}
.videolessonaddcourse{
    margin-top: 2rem;
    display: flex;
    flex-direction: row;
    justify-content: space-between;
    width: 100%;
}
/* start of video lesson */
.videoLessonText{
    display: flex;
    flex-direction: row;
    align-items: center; /* Center items vertically */
    cursor: pointer;
}
.videoLessonText p{
    margin-left: 1rem;
    font-size: clamp(1rem, 1.5vw, 2rem);
    font-weight: 600;
}
.videoLessonText box-icon{
    height: 50px;
    width: auto;
}
/* end of video lesson */

/* start of add course icon and text */
/* Base styles for the add course icon and text */
.addcourseicon {
    margin-right: 1rem;
    margin-left: auto;
    display: flex;
    flex-direction: row;
    align-items: center; /* Center items vertically */
    transition: all 0.5s ease-in;
}
.addcourseText {
    opacity: 0; /* Initially transparent */
    transform: translateX(20px); /* Position it to the right initially */
    transition: opacity 0.5s ease-in, transform 0.5s ease-in; /* Transition for appearance */
}
.addcourseText p{
    font-size: clamp(1rem, 1.5vw, 2rem);
    font-weight: 600;
}

/* Add styles for the icon */
.addcourse {
    height: 50px;
    width: auto;
    transition: transform 0.5s ease-in;
}

/* Hover effect to rotate the icon */
.addcourseicon:hover .addcourse {
    transform: rotate(90deg);
}

/* Hover effect to fade in the text and move it */
.addcourseicon:hover .addcourseText {
    opacity: 1; /* Fade in */
    transform: translateX(0); /* Move to the final position beside the icon */
}

/* Responsive styles for different screen sizes */
@media screen and (max-width: 992px) {
    .videolessonaddcourse{
        margin-top: 1rem;
    }
    .addcourse {
        margin-right: 0.5rem;
        height: 35px;
    }
    .videoLessonText box-icon{
        height: 35px;
    }
    .videoLessonText p{
        width: fit-content;
        margin-left: 0.5rem;
    }
}

@media screen and (max-width: 600px) {
    .addcourseText {
        margin-top: 0; /* Remove margin for smaller screens */
        text-align: center; /* Center text horizontally */
        transform: translateX(0); /* Reset any initial translation */
    }
}

/* Container for the Courses */
.courses-container {
    display: flex;
    flex-wrap: wrap;
    justify-content: space-around;
    padding: 20px;

}
.modal {
    display: none; /* Hidden by default */
    position: fixed;
    z-index: 1; /* Sit on top */
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    overflow: auto; /* Enable scroll if needed */
    background-color: rgba(0, 0, 0, 0.4); /* Black with opacity */
}

.modal-content {
    background-color: #fefefe;
    margin: 15% auto; /* 15% from the top and centered */
    padding: 20px;
    border: 1px solid #888;
    width: 80%; /* Could be more or less, depending on screen size */
}

.close-button {
    color: #aaa;
    float: right;
    font-size: 28px;
    font-weight: bold;
}

.close-button:hover,
.close-button:focus {
    color: black;
    text-decoration: none;
    cursor: pointer;
}

.view-more {
    color: #007bff; /* Bootstrap primary color for example */
    text-decoration: none;
}

.view-more:hover {
    text-decoration: underline;
}

/* Title and Text Styling */
h3 {
    font-size: 1.25rem;
    color: #333;
    margin: 10px 0;
}

p {
    font-size: 0.875rem;
    color: #666;
    margin: 5px 0;
}

/* Star Rating Container */
.star-rating{
    display: flex;
    flex-direction: row;
    align-items: center;
    /* justify-content: center; */
    gap: 0.5rem;
    /* margin: 10px 0; */
}
.rating-stars {
    display: flex;
    flex-direction: row;
    align-items: center;
    gap: 0.5rem;
}

/* Hide the radio inputs */
.star-input {
    display: none;
}

/* Styling for star labels */
.star-label {
    display: inline-block;
    width: 24px; /* Adjust width based on your icon size */
    height: 24px; /* Adjust height based on your icon size */
    cursor: pointer;
    fill: transparent; /* Transparent fill to allow changing star color */
    position: relative; /* Position relative for absolute positioning of half-star */
}

/* Default star color */
.star-label svg path {
    fill: #ccc; /* Color for empty star */
}

/* Style for full stars */
.star-label.full svg path {
    fill: #ffb633; /* Color for filled star */
}

/* Style for half stars */
.star-label.half svg path {
    fill: #ffb633; /* Color for filled star */
    clip-path: inset(0 50% 0 0); /* Clip the half star icon to show half only */
}

/* Positioning for half star */
.star-label.half::after {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    width: 50%; /* Adjust width to show the shade to the right */
    height: 100%;
  
}
/* Responsive Design */
@media (max-width: 768px) {
    .courses-container {
        flex-direction: column;
        align-items: center;
    }
}
/* Media query for screen sizes 762px and below */
@media (max-width: 762px) {
    .content {
        flex-direction: column; /* Stack boxes vertically */
        margin-top: 2rem;
    }
    .box1 {
        margin-top: 2rem;
        order: 2; /* Ensure box1 is on top */
        width: 100%;/* Adjust width to fit the container */
        margin-right: 0; /* Remove right margin */
        margin-bottom: 1rem; /* Add bottom margin for spacing */
        height:fit-content;
    }
    .box1-buttons{
        flex-direction: row;
        justify-content: center;
    }
    .box2 {
        order: 1; /* Ensure box2 is below box1 */
        width: 100%; /* Adjust width to fit the container */
    }
    
}
.courseCreation {
    display: none; /* Hide by default */
    justify-content: center;
    align-items: center;
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5); /* Optional: semi-transparent background */
    z-index: 999;
}

.courseCreate {
    display: flex;
    flex-direction: column;
    padding: 20px;
    border: 1px solid #ccc;
    border-radius: 10px;
    background-color: #fff;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
    width: 100%;
    max-width: 500px;
}

.courseCreate form {
    display: flex;
    flex-direction: column;
    width: 100%;
}

.courseCreate #thumbnail, 
.courseCreate #courseName, 
.courseCreate #difficulty, 
.courseCreate #NumberofLessons, 
.courseCreate #courseDescription {
    margin-top: 10px;
    margin-bottom: 5px;
    font-size: 1em;
    color: #333;
}


.courseCreate input[type="text"],
.courseCreate textarea {
    padding: 10px;
    border: 1px solid #ccc;
    border-radius: 5px;
    width: 100%;
    box-sizing: border-box;
}
.courseCreate input[type="number"] {
    padding: 10px;
    max-width: 100px;
    border: 1px solid #ccc;
    border-radius: 5px;
    width: 100%;
    box-sizing: border-box;
}
.courseCreate select {
    padding: 10px;
    border: 1px solid #ccc;
    border-radius: 5px;
    width: 100%;
    max-width: 150px;
    box-sizing: border-box;
}
.courseCreate input[type="file"] { 
    padding: 10px;
    border: 1px solid #ccc;
    border-radius: 5px;
    width: 100%;
    box-sizing: border-box;
}
.form-buttons{
    display: flex;
    gap: 1rem;
    justify-content: center;
    margin-top: 1rem;
}
.submitCourse {
    margin-top: 20px;
    padding: 10px 20px;
    background-color: #007BFF;
    color: #fff;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    transition: background-color 0.3s ease;
}

.submitCourse:hover {
    background-color: #0056b3;
}
.cancelSubmit {
    margin-top: 20px;
    padding: 10px 20px;
    background-color: #b65252;
    color: #fff;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    transition: background-color 0.3s ease;
}
/* Base styling for folder container */
.box2-content {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem; /* Reduced gap for closer spacing */
    padding: 0.5rem; /* Reduced padding for closer spacing */
    margin-top: 1rem; /* Reduced margin for closer spacing */
    width: 100%;
    height: 100%;
    max-height: 450px;
    background: #EEF0F4;
    border-radius: 10px;
    box-shadow: inset 4.8px 4.8px 2px #BBBCC0, inset -4.8px -4.8px 2px #FFFFFF;
    overflow-x: hidden;
    overflow-y: scroll;
    justify-content:center;
}

.box2-content::-webkit-scrollbar {
    width: 10px;
}

.box2-content::-webkit-scrollbar-track {
    background: #f1f1f100;
    border-radius: 10px;
}

.box2-content::-webkit-scrollbar-thumb {
    background: #3c3428;
    border-radius: 10px;
}

/* Styling for no folders */
.NoFolders {
    display: flex;
    font-size: clamp(1.5rem, 2vw, 4rem);
    color: black;
    font-weight: 600;
}

.course-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 10px;
    margin: 0.5rem; /* Reduced margin for closer spacing */
    gap: 0.5rem;
    width: 100%;
    max-width: 250px;
    height: fit-content;
    background: #EEF0F4;
    box-shadow: 0.3px 0.3px 11px #9D9EA1, -0.3px -0.3px 11px #FFFFFF;
    border-radius: 10px;
    flex: 1 1 auto; /* Allows flex items to adjust their size */
}

.course-container p {
    text-align: center;
}

.course-buttons {
    display: flex;
    gap: 0.5rem; /* Reduced gap for closer spacing */
    justify-content: center;
    margin-top: 0.5rem; /* Reduced margin for closer spacing */
}

.course-buttons button {
    padding: 5px 10px;
    border-radius: 5px;
    height: fit-content;
    width: fit-content;
    font-weight: 600;
    border: none;
}

.delete-folder {
    background-color: rgb(128, 30, 30);
    color: white;
}
.view-details {
    height: fit-content;
    width: fit-content;
    font-weight: 600;
    padding: 5px 10px;
    border-radius: 5px;
    border: none;
    background-color: #1c5c2f;
    color: #fff;
    cursor: pointer;
    transition: background-color 0.3s ease;
}

.course-container img {
    margin-top: 0.5rem;
    width: 100%;
    height: 100%;
    max-height: 100px;
    object-fit: contain;
    border-radius: 10px;
}
@media screen and (max-width: 768px) {
    .box2-content {
       
       justify-content: center;
    }
}
@media (max-width: 318px) {
    .box2-content{
       
        justify-content: center;
    }
    .course-container {
        max-width: 100%;
    }

    .course-buttons {
        flex-direction: column;
    }
}
.updateCourse-Button {
    background-color: #0056b3;
    color: white;
}
.updateButton{
    margin-top: 1rem;
    width: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
}
.updateButton button{
    padding: 10px 20px;
    border-radius: 5px;
    border: none;
    background: #3A61CF;
    color: white;
    font-weight: 600;
    box-shadow: 3.6px 3.6px 2px #B5B6B9, -3.6px -3.6px 2px #FFFFFF;
    cursor: pointer;  

}
.overlay-update-course{
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5); /* Semi-transparent background */
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000; /* Ensure it's on top of other content */
}
  /* Form Container */
  .updateCourse {
    background: #fff;
    padding: 1rem;
    border-radius: 8px;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
    width: 100%;
    max-width: 500px;
    box-sizing: border-box;
}
.closeForm{
    width: 100%;
    display: flex;
    justify-content: flex-end;
    padding: 0;
}

/* Form Elements */
.updateCourse label {
    display: block;
    margin-bottom: 0.5rem;
    font-weight: bold;
}
.updateCourse select,
.updateCourse input[type="number"],
.updateCourse input[type="text"],
.updateCourse textarea,
.updateCourse input[type="file"] {
    width: 100%;
    padding: 0.5rem;
    margin-bottom: 1rem;
    border: 1px solid #ccc;
    border-radius: 4px;
    box-sizing: border-box;
    font-size: clamp(0.8rem, 1.5vw, 1.2rem);
}
.updateCourse input[type="number"]{
    width: 100%;
    max-width: 80px;
}
.updateCourse select{
    width: 100%;
    max-width: 150px;
}
.updateCourse textarea {
    height: 100px;
    resize: vertical;
}

.updateCourseButtons {
    display: flex;
    justify-content: center;
}

.updateCourseButtons button {
    padding: 0.5rem 1rem;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 1rem;
}

.updateCourseSubmit {
    background-color: #007bff;
    color: #fff;
}

.updateCourseSubmit:hover {
    background-color: #0056b3;
}
/* Responsive Design */
@media (max-width: 600px) {
    .updateCourse {
        width: 95%;
        padding: 1rem;
    }

    .updateCourse textarea {
        height: 80px;
    }

    .updateCourseButtons button {
        font-size: 0.9rem;
    }
}

/* Overlay Styles */
.overlay-update-course {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 1000;
}

/* Popup form styles
.updateCourse {
    background-color: white;
    padding: 20px;
    box-shadow: 0px 0px 10px 0px rgba(0, 0, 0, 0.1);
    z-index: 1001;
    position: relative;
    max-width: 400px;
    width: 100%;
    border-radius: 8px;
}

.updateCourse .closeForm {
    position: absolute;
    top: 10px;
    right: 10px;
    cursor: pointer;
}

.updateCourse label {
    display: block;
    margin-top: 10px;
}

.updateCourse input,
.updateCourse textarea,
.updateCourse select {
    width: 100%;
    padding: 8px;
    margin-top: 5px;
    box-sizing: border-box;
}

.updateCourse button {
    display: block;
    margin-top: 10px;
    padding: 10px 15px;
    background-color: #007bff;
    color: white;
    border: none;
    cursor: pointer;
}

.updateCourse button:hover {
    background-color: #0056b3;
} */

</style>

