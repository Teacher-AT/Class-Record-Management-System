# Class-Record-Management-System
A Simple Python-based class record management system.
import csv
import random
import os

class Course:
    def __init__(self, course_id, name, num_quizzes):
        self.id = course_id
        self.name = name
        self.num_quizzes = num_quizzes
    def load_courses(filename):
        courses = {}
        try:
            with open(filename, newline='', encoding='utf-8-sig') as csvfile:
                reader = csv.DictReader(csvfile)
                for row in reader:
                    row = {k.strip(): v for k, v in row.items()}
                    courses[row["ID"]] = Course(row["ID"], row["Name"], int(row["NumQuizzes"]))
        except FileNotFoundError:
            pass
        return courses
    
class Score:
    def __init__(self, student_id, course_id, scores):
        self.student_id = student_id
        self.course_id = course_id
        self.scores = scores  
    
    def load_scores(filename):
        scores = []
        try:
            with open(filename, newline='', encoding='utf-8-sig') as csvfile:
                reader = csv.DictReader(csvfile)
                for row in reader:
                    row = {k.strip(): v for k, v in row.items()}
                    scores.append(Score(row["StudentID"], row["CourseID"], [int(s) for s in row["Scores"].split(',')]))
        except FileNotFoundError:
            pass
        return scores

class Student:
    def __init__(self, student_id, name):
        self.id = student_id
        self.name = name

    def load_students(filename):
        students = {}
        try:
            with open(filename, newline='', encoding='utf-8-sig') as csvfile:
                reader = csv.DictReader(csvfile)
                for row in reader:
                    row = {k.strip(): v for k, v in row.items()}
                    students[row["ID"]] = Student(row["ID"], row["Name"])
        except FileNotFoundError:
            pass
        return students

    def save_students(filename, students):
        with open(filename, mode='w', newline='') as csvfile:
            writer = csv.DictWriter(csvfile, fieldnames=["ID", "Name"])
            writer.writeheader()
            for student in students.values():
                writer.writerow({"ID": student.id, "Name": student.name})

class ClassRecordSystem:
    def __init__(self):
        self.students = {}
        self.courses = {}
        self.scores = []
        self.undo_stack = []

    def clear_screen(self):
        os.system('cls' if os.name == 'nt' else 'clear')

    def upload_csv(self):
        filename = input("Enter CSV filename to upload: ").strip()
        self.students = {}
        self.courses = {}
        self.scores = []
        try:
            with open(filename, newline='', encoding='utf-8-sig') as csvfile:
                reader = csv.DictReader(csvfile, delimiter='|')
                quiz_columns = [col for col in reader.fieldnames if col.startswith("Quiz #")]
                for row in reader:
                    student_id = row["ID"].strip()
                    student_name = row["Name"].strip()
                    course_name = row["Course"].strip()
                    scores = []
                    for q in quiz_columns:
                        val = row[q].strip()
                        if val != "":
                            scores.append(int(val))
                    if student_id not in self.students:
                        self.students[student_id] = Student(student_id, student_name)
                    if course_name not in self.courses:
                        self.courses[course_name] = Course(course_name, course_name, len(quiz_columns))
                    self.scores.append(Score(student_id, course_name, scores))
            print(f"Loaded {len(self.students)} students, {len(self.courses)} courses, and {len(self.scores)} scores from {filename}.")
        except FileNotFoundError:
            print("File not found.")

    def save_csv(self):
        filename = input("Enter CSV filename to save: ").strip()
        max_quizzes = max((c.num_quizzes for c in self.courses.values()), default=0)
        quiz_headers = [f"Quiz #{i+1}" for i in range(max_quizzes)]
        fieldnames = ["ID", "Name", "Course"] + quiz_headers

        with open(filename, mode='w', newline='', encoding='utf-8-sig') as csvfile:
            writer = csv.DictWriter(csvfile, fieldnames=fieldnames, delimiter='|')
            writer.writeheader()
            for score in self.scores:
                student = self.students.get(score.student_id)
                course = self.courses.get(score.course_id)
                if student and course:
                    row = {
                        "ID": student.id,
                        "Name": student.name,
                        "Course": course.name
                    }
                    for i in range(max_quizzes):
                        key = f"Quiz #{i+1}"
                        row[key] = str(score.scores[i]) if i < len(score.scores) else ""
                    writer.writerow(row)
        print(f"Saved {len(self.scores)} records to {filename}.")

    def display_students(self):
        print("ID | Name")
        if not self.students:
            print("[No students found]")
        else:
            for s in self.students.values():
                print(f"{s.id} | {s.name}")

    def display_courses(self):
        if not self.courses:
            print("[No courses found]")
        else:
            for idx, course_name in enumerate(self.courses, 1):
                print(f"{idx}. {course_name}")

    def add_student(self):
        name = input("Enter student name: ").strip()
        if any(s.name.lower() == name.lower() for s in self.students.values()):
            print(f"Student '{name}' already exists.")
            return
        if not self.courses:
            print("No courses available. Please add a course first.")
            return
        print("Available courses:")
        course_name = list(self.courses.keys())
        for idx, cname in enumerate(course_name, 1):
            course = self.courses[cname]
            print(f"{idx}. {course.name}")
        try:
            course_choice = int(input("Select course number: "))
            if not (1 <= course_choice <= len(course_name)):
                print("Invalid course selection.")
                return
        except ValueError:
            print("Invalid input.")
            return
        course_name = course_name[course_choice - 1]
        course = self.courses[course_name]
        scores = []
        for i in range(course.num_quizzes):
            while True:
                try:
                    score = int(input(f"Enter score for Quiz #{i+1}: "))
                    scores.append(score)
                    break
                except ValueError:
                    print("Please enter a valid integer score.")
        student_id = str(random.randint(1000, 9999))
        self.students[student_id] = Student(student_id, name)
        self.scores.append(Score(student_id, course_name, scores))
        self.undo_stack.append(('add_student', student_id, course_name, scores))
        print(f"Student '{name}' added with ID {student_id} to course '{course.name}'.")
        if scores:
            avg= sum(scores) / len(scores)
            print(f"Average score for: {avg:.2f}")
        self.display_full_records()

    def update_student(self):
        self.display_students()
        student_id = input("Enter student ID to update: ").strip()
        if student_id in self.students:
            old_name = self.students[student_id].name
            new_name = input("Enter new name: ").strip()
            self.students[student_id].name = new_name
            self.undo_stack.append(('update_student', student_id, old_name))
            print("Student updated.")
            self.display_full_records()
        else:
            print("Student ID not found.")

    def delete_student(self):
        self.display_students()
        student_id = input("Enter student ID to delete: ").strip()
        if student_id in self.students:
            student_obj = self.students[student_id]
            self.undo_stack.append(('delete_student', student_id, student_obj.name))
            del self.students[student_id]
            print("Student deleted.")
            self.display_full_records()
        else:
            print("Student ID not found.")

    def add_course(self):
        name = input("Enter course name: ").strip()
        if name in self.courses:
            print(f"Course '{name}' already exists.")
            return
        num_quizzes = int(input("Enter number of quizzes: "))
        self.courses[name] = Course(name, name, num_quizzes)
        print(f"Course '{name}' added.")

    def update_course(self):
        self.display_courses()
        course_name = input("Enter course ID to update: ").strip()
        if course_name in self.courses:
            old_name = self.courses[course_name].name
            old_num_quizzes = self.courses[course_name].num_quizzes
            new_name = input("Enter new course name: ").strip()
            num_quizzes = int(input("Enter new number of quizzes: "))
            self.courses[course_name].name = new_name
            self.courses[course_name].num_quizzes = num_quizzes
            self.undo_stack.append(('update_course', course_name, old_name, old_num_quizzes))
            print("Course updated.")
        else:
            print("Course ID not found.")

    def delete_course(self):
        self.display_courses()
        course_name = input("Enter course ID to delete: ").strip()
        if course_name in self.courses:
            course_obj = self.courses[course_name]
            self.undo_stack.append(('delete_course', course_name, course_obj.name, course_obj.num_quizzes))
            del self.courses[course_name]
            print("Course deleted.")
        else:
            print("Course ID not found.")

    def undo(self):
        if not self.undo_stack:
            print("Nothing to undo.")
            return
        action = self.undo_stack.pop()
        if action[0] == 'add_student':
            student_id = action[1]
            if student_id in self.students:
                del self.students[student_id]
                print(f"Undo: Added student with ID {student_id} removed.")
        elif action[0] == 'update_student':
            student_id, old_name = action[1], action[2]
            if student_id in self.students:
                self.students[student_id].name = old_name
                print(f"Undo: Student ID {student_id} name reverted to '{old_name}'.")
        elif action[0] == 'delete_student':
            student_id, name = action[1], action[2]
            self.students[student_id] = Student(student_id, name)
            print(f"Undo: Deleted student '{name}' with ID {student_id} restored.")
        elif action[0] == 'add_course':
            course_name = action[1]
            if course_name in self.courses:
                del self.courses[course_name]
                print(f"Undo: Added course removed.")
        elif action[0] == 'update_course':
            course_name, old_name, old_num_quizzes = action[1], action[2], action[3]
            if course_name in self.courses:
                self.courses[course_name].name = old_name
                self.courses[course_name].num_quizzes = old_num_quizzes
                print(f"Undo: Course ID {course_name} reverted to '{old_name}' with {old_num_quizzes} quizzes.")
        elif action[0] == 'delete_course':
            course_name, name, num_quizzes = action[1], action[2], action[3]
            self.courses[course_name] = Course(course_name, name, num_quizzes)
            print(f"Undo: Deleted course '{name}' ")

    def display_full_records(self):
        max_quizzes = max((c.num_quizzes for c in self.courses.values()), default=0)
        quiz_headers = [f"Quiz #{i+1}" for i in range(max_quizzes)]
        print("ID | Name | Course | " + " | ".join(quiz_headers))
        print("-" * (40 + 8 * max_quizzes))
        found = False
        for score in self.scores:
            student = self.students.get(score.student_id)
            course = self.courses.get(score.course_id)
            if student and course:
                found = True
                scores_display = [str(s) for s in score.scores] + [""] * (max_quizzes - len(score.scores))
                print(f"{student.id} | {student.name} | {course.name} | " + " | ".join(scores_display))
        if not found:
            print("[No records found]")

    def manage_students(self):
        while True:
            print("\n--- Manage Students ---")
            print("1. Add Student")
            print("2. Update Student")
            print("3. Delete Student")
            print("4. Back to Main Menu")
            choice = input("Choose an option: ").strip()
            if choice == "1":
                self.add_student()
            elif choice == "2":
                self.update_student()
            elif choice == "3":
                self.delete_student()
            elif choice == "4":
                break
            else:
                print("Invalid choice.")
            input("Press Enter to continue...")

    def manage_courses(self):
        while True:
            print("\n--- Manage Courses ---")
            print("1. Add Course")
            print("2. Update Course")
            print("3. Delete Course")
            print("4. Back to Main Menu")
            choice = input("Choose an option: ").strip()
            if choice == "1":
                self.add_course()
            elif choice == "2":
                self.update_course()
            elif choice == "3":
                self.delete_course()
            elif choice == "4":
                break
            else:
                print("Invalid choice.")
            input("Press Enter to continue...")

    def main_menu(self):
        while True:
            self.clear_screen()
            print("\n=== MAIN MENU ===")
            print("1. Upload CSV File")
            print("2. Manage Students (Add/Update/Delete)")
            print("3. Manage Courses (Add/Update/Delete)")
            print("4. Undo Last Action")
            print("5. Display Students")
            print("6. Display Courses")
            print("7. Display Full Records")
            print("8. Save to CSV")
            print("9. Exit")
            choice = input("Choose an option: ").strip()

            if choice == "1":
                self.upload_csv()
            elif choice == "2":
                self.manage_students()
            elif choice == "3":
                self.manage_courses()
            elif choice == "4":
                self.undo()
            elif choice == "5":
                self.display_students()
            elif choice == "6":
                self.display_courses()
            elif choice == "7":
                self.display_full_records()
            elif choice == "8":
                self.save_csv()
            elif choice == "9":
                print("Exiting...")
                break
            else:
                print("Invalid choice.")
            input("Press Enter to continue...")

if __name__ == "__main__":
    crs = ClassRecordSystem()
    crs.main_menu()
