Read README.md before anything else. It explains the app structure, traces two example call chains, and lists the five open issues with their affected service files. This context shapes everything in the milestones that follow.


Read through the main files before looking at any issue. AI tools are genuinely useful for this phase. Here are prompting patterns that work well for codebase orientation:

File summary: Give the AI the contents of a service file and ask "What is this module responsible for? What are its main functions and what does each one do?"
Data flow trace: Ask "Given this services/ directory, trace how a song gets added to a user's feed — which functions are called and in what order?" (paste the relevant files as context)
Function explanation: Give the AI a function you don't understand and ask "Walk me through what this function does step by step, including what it returns and what could cause it to return an unexpected value."
Take notes as you go. The goal of this phase is to build a mental model of how the app is organized — not to find bugs yet.


Write your codebase map in submission.md (create this file in the root of your repo — this is your submission doc for the whole project). Cover at minimum: the main files and what each one does, the data flow for at least one feature (e.g., how sharing a song triggers a notification), and any patterns you notice in how the app is organized.

📋 What a useful codebase map looks like
A weak map just lists files by name without saying what they actually do or how they connect:

"The app has app.py, models.py, routes/, and services/. The routes handle the endpoints and the services do the logic."

A strong map names the responsibility of each piece and traces at least one real data flow:

"models.py defines 5 SQLAlchemy models: User, Song, Playlist, PlaylistSong, and Notification. The PlaylistSong table is a join table that adds an order column — songs in a playlist have an explicit position, not just insertion order.

Data flow — user rates a song: POST /songs/<id>/rate in routes/songs.py calls notification_service.notify_song_rated(). That function creates a Notification record for the song's original sharer. There's no separate rating model — the rating is stored directly on the Song.

Pattern I noticed: every route delegates immediately to a service function. The routes do input parsing and response formatting; all business logic lives in services/."

The strong version proves you read the code. The weak version could have been written by someone who just looked at the file tree.


Read all five issue descriptions before choosing which three to fix. Some bugs share similar root cause patterns — knowing all five before starting helps you navigate more efficiently.

