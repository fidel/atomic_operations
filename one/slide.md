!SLIDE

# Atomic operations

Szymon Fiedler | @fidelski | github.com/fidel

!SLIDE

![You're doing it wrong!](wrong.jpg)

probably

!SLIDE smaller bullets incremental

# Let's ensure that only 30 students can sign up for course

    @@@ ruby
    @course = Course.find(1)
    if @course.num_students < 30
      @course.course_students.create!(:student_id => 101)
      @course.num_students += 1
      @course.save!
    else
      # course is full
    end

* 32 students in 30 person class. WTF?!

!SLIDE smaller bullets incremental

# Let's use ActiveRecord locking

    @@@ ruby
    @course = Course.find(1, :lock => true)
    if @course.num_students < 30
      # ...

* If you need high concurency, you're screwed

!SLIDE smaller bullets incremental

# Let's remove separate counter

    @@@ ruby
    @course = Course.find(1)
    if @course.course_students.count < 30
      @course.course_students.create!(:student_id => 101)
    else
      # course is full
    end

* Nope, it still won't work

!SLIDE bullets incremental smaller

# When race conidition arises?

* when there's difference between evaluating and altering a value


* the more lines of code between operation
* the higher user count
* the bigger the window of opportunity for other clients to get data in an incosistent state

!SLIDE bullets incremental smaller

# Probably race condition, but seems legit

    @@@ ruby
    @user = User.find(params[:id])
    @post = Post.create(:user_id => @user.id,
      :title => "Whattup")
    @user.total_posts += 1  # update my post count

!SLIDE bullets incremental smaller

# However this would be problematic

    @@@ ruby
    @blog = Blog.find(params[:id])
    @post = Post.create(:blog_id => @blog.id,
      :title => "Whattup")
    @blog.total_posts += 1  # update post count across all users    

* As multiple users could be adding posts concurrently.

!SLIDE bullets incremental smaller

## You can increment counters atomically in RDBMS but not return them

    @@@ sql
    update users set total_posts = total_posts + 1 where id = 372

!SLIDE bullets incremental smaller

## You can use AR increment_counter class method

    @@@ ruby
    @blog = Blog.find(params[:id])
    Blog.increment_counter :total_posts, @blog.id
    if @blog.total_posts == 1000
      # the 1000th poster - award them a gold star!

* It solves problem a half
* Your object is no longer in sync with the DB
* The DB says 1000, but your @blog object still says 999, and the right person doesn’t get their gold star.
* Sad faces all around.

!SLIDE bullets incremental smaller

# But there's a better way

* Any operation that can alter a value must return that value in the same operation for it to be atomic
* There are few systems that support "increment and return"
* Redis
* Oracle sequences

!SLIDE bullets incremental smaller

# Use cases

* Ensuring there are no more than 30 students in a course
* Getting more than 2 but less than 6 people in a game
* Keeping a chat room to a max of 50 people
* Correctly recording the total number of blog posts

!SLIDE bullets incremental smaller

# Best way

* A counter you base logic on (eg, slots_taken)
* A counter users see (eg, current_students)

* Why?

* You increment logic counter first before checking it to address race condition
* But that's not a problem

!SLIDE smaller

# Course example

    @@@ ruby
    class Course < ActiveRecord::Base
      include Redis::Objects

      counter :slots_taken
      counter :current_students
    end

    
!SLIDE smaller bullets incremental

    @@@ ruby
    @course = Course.find(1)
    @course.slots_taken.increment do |val|
      if val <= @course.max_students
        @course.course_students.create!(:student_id => 101)
        @course.current_students.increment
      end
    end

* Race condition free
* Because we're checking direct result of the increment operation against a value


!SLIDE smaller bullets

# Source

* http://nateware.com/2010/02/18/an-atomic-rant/

!SLIDE smaller

# Thanks!
