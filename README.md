from flask import Flask, request, jsonify, send_from_directory
from werkzeug.security import generate_password_hash, check_password_hash
from models import db, User, Video, Interaction
import os

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///database.db'
app.config['UPLOAD_FOLDER'] = 'uploads'
app.config['SECRET_KEY'] = 'supersecretkey'

db.init_app(app)

if not os.path.exists('uploads'):
    os.makedirs('uploads')

@app.before_first_request
def create_tables():
    db.create_all()

# -------- USER AUTH --------
@app.route('/register', methods=['POST'])
def register():
    data = request.json
    hashed = generate_password_hash(data['password'])
    user = User(username=data['username'], password=hashed)
    db.session.add(user)
    db.session.commit()
    return jsonify({'status':'ok'})

@app.route('/login', methods=['POST'])
def login():
    data = request.json
    user = User.query.filter_by(username=data['username']).first()
    if user and check_password_hash(user.password, data['password']):
        return jsonify({'status':'ok', 'user_id': user.id})
    return jsonify({'status':'fail'})

# -------- VIDEO UPLOAD --------
@app.route('/upload', methods=['POST'])
def upload_video():
    file = request.files['video']
    tags = request.form.get('tags','')  # optional tags
    path = os.path.join(app.config['UPLOAD_FOLDER'], file.filename)
    file.save(path)
    video = Video(filename=file.filename, tags=tags)
    db.session.add(video)
    db.session.commit()
    return jsonify({'status':'ok'})

# -------- AI-STYLE FEED --------
@app.route('/videos/<int:user_id>')
def ai_feed(user_id):
    """
    Returns a ranked video feed based on:
    - Videos user previously liked
    - Videos with similar tags
    - Most liked videos overall
    """
    # Step 1: Get user interactions
    interactions = Interaction.query.filter_by(user_id=user_id).all()
    liked_ids = [i.video_id for i in interactions if i.liked]

    # Step 2: Get tags of liked videos
    liked_tags = []
    for vid_id in liked_ids:
        video = Video.query.get(vid_id)
        if video and video.tags:
            liked_tags += video.tags.split(',')

    # Step 3: Score all videos
    videos = Video.query.all()
    scored_videos = []

    for v in videos:
        score = v.likes * 1.0  # base score
        # boost score if user liked it before
        if v.id in liked_ids:
            score += 50
        # boost if tags match previous likes
        if v.tags:
            tags = v.tags.split(',')
            score += 5 * sum(tag in liked_tags for tag in tags)
        scored_videos.append((score, v))

    # Step 4: Sort videos by score descending
    scored_videos.sort(key=lambda x: x[0], reverse=True)

    # Step 5: Return JSON
    return jsonify([{'id': v.id, 'url': f'/uploads/{v.filename}', 'likes': v.likes} for _, v in scored_videos])

# -------- LIKE VIDEO --------
@app.route('/like/<int:user_id>/<int:video_id>', methods=['POST'])
def like_video(user_id, video_id):
    video = Video.query.get(video_id)
    video.likes += 1
    db.session.commit()
    # record interaction
    interaction = Interaction(user_id=user_id, video_id=video_id, liked=True)
    db.session.add(interaction)
    db.session.commit()
    return jsonify({'likes': video.likes})

# -------- COMMENT --------
@app.route('/comment/<int:video_id>', methods=['POST'])
def comment(video_id):
    data = request.json
    # For simplicity, comments not used in recommendation yet
    return jsonify({'status':'ok'})

# -------- VIDEO SERVE --------
@app.route('/uploads/<filename>')
def serve_video(filename):
    return send_from_directory(app.config['UPLOAD_FOLDER'], filename)

# -------- RUN --------
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
