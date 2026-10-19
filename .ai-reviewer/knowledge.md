# personalblog reviewer notes

## Architecture
A small Flask server-rendered blog application implemented primarily in `app.py`. Flask-SQLAlchemy models (`User`, `Post`), Flask-Login authentication, route handlers, and application startup are all colocated in that file; HTML is rendered from templates under the standard Flask templates directory. SQLite is the configured persistence layer (`sqlite:///blog.db`).

## Conventions
- Use Flask route functions with `render_template`, `redirect`, `url_for`, and `flash`; route handlers directly perform queries and commits, as shown throughout `app.py`.
- Authentication-protected routes use `@login_required`, including `/dashboard`, `/create`, `/edit/<int:post_id>`, `/delete/<int:post_id>`, and `/logout` (`app.py`).
- Ownership must be checked explicitly before modifying a post. Both `edit_post` and `delete_post` compare `post.user_id` with `current_user.id` and redirect unauthorized users to the dashboard (`app.py`).
- User passwords are stored as Werkzeug hashes created with `generate_password_hash(..., method='pbkdf2:sha256', salt_length=8)` and verified with `check_password_hash` (`app.py`).
- Database mutations follow the pattern of adding/deleting or modifying a model, then calling `db.session.commit()`; successful operations flash a message and redirect.
- Post listings are newest-first on the home page using `Post.query.order_by(Post.date_posted.desc()).all()` (`app.py`); the dashboard filters by the logged-in user.
- Dynamic identifiers are expressed as typed route parameters such as `/post/<int:post_id>`, and missing records are handled with `get_or_404` (`app.py`).
- The application creates database tables at startup inside `app.app_context()` and only runs the development server under `if __name__ == '__main__'` (`app.py`).

## Intentional non-standard choices
- The project is intentionally compact and monolithic: models, authentication callbacks, routes, and startup configuration remain in `app.py` rather than being split into blueprints or packages.
- SQLite and `db.create_all()` are used for the application’s lightweight persistence setup; there is no visible migration framework.

## Watch out for
- Do not introduce or preserve production deployment with the literal `SECRET_KEY = 'your_secret_key'` or `app.run(debug=True)` (`app.py`); these expose security and debugging risks.
- Flag missing input validation in `/register`, `/login`, `/create`, and `/edit`: empty values can reach the database or password-hashing code, and duplicate usernames can raise an integrity error (`app.py`).
- Flag state-changing actions exposed as GET routes, especially `/delete/<int:post_id>` and `/logout`; these should normally use POST with CSRF protection.
- Check authorization on every future post mutation. The existing edit/delete handlers enforce ownership, but a new mutation route must not rely only on `@login_required`.
- Note that `user_id` lacks `nullable=False` and there is no declared ORM relationship between `User` and `Post` (`app.py`); changes should preserve or deliberately improve this data-integrity model.
- Schema changes cannot be safely managed by `db.create_all()` alone; production changes require an explicit migration strategy.