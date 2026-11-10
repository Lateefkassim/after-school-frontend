<script>
// change this to the render.com address after deploying
const API = 'https://m01142449-after-school.onrender.com'

export default {
  data() {
    return {
      imageUrl: API + '/images/',
      lessons: [],
      teachers: [],
      cart: [],
      showCart: false,
      searchText: '',
      teacherId: '',
      sortBy: 'topic',
      sortOrder: 'asc',
      name: '',
      phone: '',
      message: '',
      error: '',
    }
  },

  computed: {
    sortedLessons() {
      const list = [...this.lessons]
      list.sort((a, b) => {
        let result
        if (this.sortBy === 'price' || this.sortBy === 'space') {
          result = a[this.sortBy] - b[this.sortBy]
        } else {
          result = a[this.sortBy].localeCompare(b[this.sortBy])
        }
        if (this.sortOrder === 'desc') {
          result = -result
        }
        return result
      })
      return list
    },

    cartCount() {
      let count = 0
      for (const item of this.cart) {
        count = count + item.spaces
      }
      return count
    },

    totalPrice() {
      let total = 0
      for (const item of this.cart) {
        total = total + item.price * item.spaces
      }
      return total
    },

    nameValid() {
      return /^[A-Za-z ]+$/.test(this.name)
    },

    phoneValid() {
      return /^[0-9]+$/.test(this.phone)
    },
  },

  mounted() {
    this.getLessons()
    this.getTeachers()
  },

  methods: {
    // fetch functions
    getLessons() {
      fetch(API + '/lessons')
        .then((res) => res.json())
        .then((data) => this.showLessons(data))
        .catch(() => (this.error = 'Could not load the lessons'))
    },

    getTeachers() {
      fetch(API + '/teachers')
        .then((res) => res.json())
        .then((data) => (this.teachers = data))
        .catch(() => (this.error = 'Could not load the teachers'))
    },

    search() {
      this.teacherId = ''
      if (this.searchText === '') {
        this.getLessons()
        return
      }
      fetch(API + '/search?q=' + encodeURIComponent(this.searchText))
        .then((res) => res.json())
        .then((data) => this.showLessons(data))
        .catch(() => (this.error = 'Search failed'))
    },

    chooseTeacher() {
      this.searchText = ''
      if (this.teacherId === '') {
        this.getLessons()
        return
      }
      fetch(API + '/teachers/' + this.teacherId + '/lessons')
        .then((res) => res.json())
        .then((data) => this.showLessons(data))
        .catch(() => (this.error = 'Could not load the teacher lessons'))
    },

    // takes the spaces that are already in the cart off the lessons
    showLessons(data) {
      this.error = ''
      for (const lesson of data) {
        const item = this.cart.find((i) => i.lessonId === lesson._id)
        if (item) {
          lesson.space = lesson.space - item.spaces
        }
      }
      this.lessons = data
    },

    addToCart(lesson) {
      if (lesson.space === 0) {
        return
      }
      this.message = ''
      const item = this.cart.find((i) => i.lessonId === lesson._id)
      if (item) {
        item.spaces++
      } else {
        this.cart.push({
          lessonId: lesson._id,
          topic: lesson.topic,
          location: lesson.location,
          price: lesson.price,
          spaces: 1,
          startSpace: lesson.space,
        })
      }
      lesson.space--
    },

    removeFromCart(item) {
      const lesson = this.lessons.find((l) => l._id === item.lessonId)
      if (lesson) {
        lesson.space = lesson.space + item.spaces
      }
      this.cart.splice(this.cart.indexOf(item), 1)
      if (this.cart.length === 0) {
        this.showCart = false
      }
    },

    checkout() {
      const items = this.cart.map((item) => ({ lessonId: item.lessonId, spaces: item.spaces }))
      const headers = { 'Content-Type': 'application/json' }

      fetch(API + '/orders', {
        method: 'POST',
        headers: headers,
        body: JSON.stringify({ name: this.name, phone: this.phone, items: items }),
      })
        .then((res) => {
          if (!res.ok) {
            throw new Error('Order failed')
          }
          return res.json()
        })
        .then(() => {
          const updates = this.cart.map((item) =>
            fetch(API + '/lessons/' + item.lessonId, {
              method: 'PUT',
              headers: headers,
              body: JSON.stringify({ space: item.startSpace - item.spaces }),
            })
          )
          return Promise.all(updates)
        })
        .then(() => {
          this.message = 'Thank you ' + this.name + ', your order has been submitted!'
          this.cart = []
          this.name = ''
          this.phone = ''
          this.showCart = false
          this.getLessons()
        })
        .catch(() => (this.error = 'Could not submit the order'))
    },
  },
}
</script>

<template>
  <header class="topbar">
    <div class="topbar-inner">
      <h1><i class="fa-solid fa-graduation-cap"></i> After School Classes</h1>
      <button class="cart-button" :disabled="cart.length === 0" @click="showCart = !showCart">
        <i class="fa-solid fa-cart-shopping"></i>
        {{ showCart ? 'Back to lessons' : 'Cart' }}
        <span v-if="!showCart" class="badge">{{ cartCount }}</span>
      </button>
    </div>
  </header>

  <main class="page">
    <p v-if="message" class="message success"><i class="fa-solid fa-circle-check"></i> {{ message }}</p>
    <p v-if="error" class="message failed"><i class="fa-solid fa-circle-exclamation"></i> {{ error }}</p>

    <!-- lessons page -->
    <div v-if="!showCart">
      <div class="controls">
        <div class="search-box">
          <i class="fa-solid fa-magnifying-glass"></i>
          <input v-model="searchText" @input="search" placeholder="Search lessons" />
        </div>

        <select v-model="teacherId" @change="chooseTeacher">
          <option value="">All teachers</option>
          <option v-for="teacher in teachers" :key="teacher._id" :value="teacher._id">
            {{ teacher.firstName }} {{ teacher.lastName }}
          </option>
        </select>

        <select v-model="sortBy">
          <option value="topic">Sort by subject</option>
          <option value="location">Sort by location</option>
          <option value="price">Sort by price</option>
          <option value="space">Sort by spaces</option>
        </select>

        <select v-model="sortOrder">
          <option value="asc">Ascending</option>
          <option value="desc">Descending</option>
        </select>
      </div>

      <p v-if="lessons.length === 0" class="empty">No lessons found.</p>

      <div class="lessons">
        <div class="lesson" v-for="lesson in sortedLessons" :key="lesson._id">
          <img :src="imageUrl + lesson.image" :alt="lesson.topic" />

          <div class="lesson-body">
            <div class="lesson-title">
              <h2><i :class="lesson.icon"></i> {{ lesson.topic }}</h2>
              <span class="price">£{{ lesson.price }}</span>
            </div>

            <p class="location"><i class="fa-solid fa-location-dot"></i> {{ lesson.location }}</p>

            <p class="spaces" :class="{ low: lesson.space > 0 && lesson.space <= 2, full: lesson.space === 0 }">
              <i class="fa-solid fa-users"></i> Spaces: {{ lesson.space }}
            </p>

            <button :disabled="lesson.space === 0" @click="addToCart(lesson)">Add to Cart</button>
          </div>
        </div>
      </div>
    </div>

    <!-- cart page with checkout -->
    <div v-else class="cart-page">
      <div class="card">
        <h2>Your Cart</h2>

        <div class="cart-item" v-for="item in cart" :key="item.lessonId">
          <div class="cart-info">
            <strong>{{ item.topic }}</strong>
            <span>{{ item.location }}</span>
          </div>
          <span>{{ item.spaces }} x £{{ item.price }}</span>
          <button class="remove" @click="removeFromCart(item)">Remove</button>
        </div>

        <p class="total">Total: £{{ totalPrice }}</p>
      </div>

      <div class="card">
        <h2>Checkout</h2>

        <label for="name">Name</label>
        <input id="name" v-model="name" placeholder="Your name" />
        <p class="error" v-if="name !== '' && !nameValid">Name must be letters only</p>

        <label for="phone">Phone</label>
        <input id="phone" v-model="phone" placeholder="Your phone number" />
        <p class="error" v-if="phone !== '' && !phoneValid">Phone must be numbers only</p>

        <button class="checkout" :disabled="!nameValid || !phoneValid" @click="checkout">Checkout</button>
      </div>
    </div>
  </main>
</template>

<style>
:root {
  --blue: #2563eb;
  --blue-dark: #1d4ed8;
  --background: #f1f5f9;
  --text: #1e293b;
  --muted: #64748b;
  --border: #e2e8f0;
  --red: #dc2626;
  --green: #16a34a;
}

body {
  margin: 0;
  font-family: -apple-system, 'Segoe UI', Roboto, Arial, sans-serif;
  background: var(--background);
  color: var(--text);
}

/* top bar */
.topbar {
  background: white;
  border-bottom: 1px solid var(--border);
  position: sticky;
  top: 0;
  z-index: 10;
}

.topbar-inner {
  max-width: 1100px;
  margin: 0 auto;
  padding: 14px 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

h1 {
  margin: 0;
  font-size: 22px;
}

h1 i {
  color: var(--blue);
}

.page {
  max-width: 1100px;
  margin: 0 auto;
  padding: 24px 20px;
}

/* buttons */
button {
  background: var(--blue);
  color: white;
  border: none;
  padding: 10px 16px;
  border-radius: 8px;
  cursor: pointer;
  font-size: 15px;
  font-weight: 600;
}

button:hover:not(:disabled) {
  background: var(--blue-dark);
}

button:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}

.badge {
  background: white;
  color: var(--blue);
  border-radius: 20px;
  padding: 1px 8px;
  margin-left: 4px;
  font-size: 13px;
}

/* messages */
.message {
  padding: 12px 16px;
  border-radius: 8px;
  margin: 0 0 20px;
}

.success {
  background: #dcfce7;
  color: #166534;
}

.failed {
  background: #fee2e2;
  color: #991b1b;
}

/* search, teacher and sort */
.controls {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  background: white;
  padding: 14px;
  border-radius: 12px;
  border: 1px solid var(--border);
  margin-bottom: 24px;
}

.search-box {
  position: relative;
  flex: 1;
  min-width: 200px;
}

.search-box i {
  position: absolute;
  left: 12px;
  top: 12px;
  color: var(--muted);
}

.search-box input {
  width: 100%;
  box-sizing: border-box;
  padding-left: 36px;
}

input,
select {
  padding: 10px 12px;
  border: 1px solid var(--border);
  border-radius: 8px;
  font-size: 15px;
  background: white;
  color: var(--text);
}

input:focus,
select:focus {
  outline: none;
  border-color: var(--blue);
}

.empty {
  text-align: center;
  color: var(--muted);
  padding: 40px 0;
}

/* lesson cards */
.lessons {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
  gap: 20px;
}

.lesson {
  background: white;
  border-radius: 12px;
  border: 1px solid var(--border);
  overflow: hidden;
  transition: transform 0.2s, box-shadow 0.2s;
}

.lesson:hover {
  transform: translateY(-4px);
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.1);
}

.lesson img {
  width: 100%;
  height: 150px;
  object-fit: cover;
  display: block;
}

.lesson-body {
  padding: 16px;
}

.lesson-title {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.lesson-title h2 {
  margin: 0;
  font-size: 19px;
}

.lesson-title h2 i {
  color: var(--blue);
}

.price {
  font-weight: 700;
  font-size: 18px;
  color: var(--green);
}

.location {
  color: var(--muted);
  margin: 8px 0;
}

.spaces {
  color: var(--green);
  font-weight: 600;
  margin: 0 0 14px;
}

.spaces.low {
  color: #d97706;
}

.spaces.full {
  color: var(--red);
}

.lesson button {
  width: 100%;
}

/* cart page */
.cart-page {
  display: grid;
  grid-template-columns: 2fr 1fr;
  gap: 20px;
  align-items: start;
}

.card {
  background: white;
  padding: 20px;
  border-radius: 12px;
  border: 1px solid var(--border);
}

.card h2 {
  margin: 0 0 16px;
  font-size: 20px;
}

.cart-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  padding: 12px 0;
  border-bottom: 1px solid var(--border);
}

.cart-info {
  display: flex;
  flex-direction: column;
  flex: 1;
}

.cart-info span {
  color: var(--muted);
  font-size: 14px;
}

.remove {
  background: var(--red);
  padding: 6px 12px;
  font-size: 14px;
}

.remove:hover:not(:disabled) {
  background: #b91c1c;
}

.total {
  text-align: right;
  font-weight: 700;
  font-size: 18px;
  margin: 16px 0 0;
}

/* checkout form */
.card label {
  display: block;
  font-weight: 600;
  margin-bottom: 6px;
}

.card input {
  width: 100%;
  box-sizing: border-box;
  margin-bottom: 14px;
}

.error {
  color: var(--red);
  font-size: 14px;
  margin: -8px 0 14px;
}

.checkout {
  width: 100%;
}

/* small screens */
@media (max-width: 700px) {
  .cart-page {
    grid-template-columns: 1fr;
  }

  h1 {
    font-size: 18px;
  }
}
</style>
