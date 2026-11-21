# After School Classes - Frontend

This is the Vue.js app for my after school classes app (CST3144). You can look at the lessons, search and sort them, add them to a cart and check out. It gets its data from my Express server using `fetch`.

## What it does

- Shows the lessons with a picture, price, location and spaces left
- Sort by subject, location, price or spaces, ascending or descending
- Search (done on the server) and filter by teacher
- Add lessons to a cart and remove them again
- Checkout with a name (letters only) and phone (numbers only)

## How to run it

1. Run `npm install`
2. Run `npm run dev`
3. Open the link it shows (usually http://localhost:5173)

The server address is the `API` line at the top of `src/App.vue`.

## Links

- Vue.js app repository: https://github.com/Lateefkassim/after-school-frontend
- Vue.js app on GitHub Pages: https://lateefkassim.github.io/after-school-frontend/
- Express.js app repository: https://github.com/Lateefkassim/after-school-backend
- Lessons route on Render: https://m01142449-after-school.onrender.com/lessons
