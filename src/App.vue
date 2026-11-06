<script>
// change this to the render.com address after deploying
const API = 'http://localhost:3000'

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
        .catch(() => (this.message = 'Could not load the lessons'))
    },

    getTeachers() {
      fetch(API + '/teachers')
        .then((res) => res.json())
        .then((data) => (this.teachers = data))
        .catch(() => (this.message = 'Could not load the teachers'))
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
        .catch(() => (this.message = 'Search failed'))
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
        .catch(() => (this.message = 'Could not load the teacher lessons'))
    },

    // takes the spaces that are already in the cart off the lessons
    showLessons(data) {
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
        .catch(() => (this.message = 'Could not submit the order'))
    },
  },
}
</script>

<template>
  <div class="page">
    <header>
      <h1><i class="fa-solid fa-graduation-cap"></i> After School Classes</h1>
      <button class="cart-button" :disabled="cart.length === 0" @click="showCart = !showCart">
        <i class="fa-solid fa-cart-shopping"></i>
        {{ showCart ? 'Back to lessons' : 'Cart (' + cartCount + ')' }}
      </button>
    </header>

    <p v-if="message" class="message">{{ message }}</p>

    <!-- lessons page -->
    <div v-if="!showCart">
      <div class="controls">
        <input v-model="searchText" @input="search" placeholder="Search lessons" />

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

      <p v-if="lessons.length === 0">No lessons found.</p>

      <div class="lessons">
        <div class="lesson" v-for="lesson in sortedLessons" :key="lesson._id">
          <img :src="imageUrl + lesson.image" :alt="lesson.topic" />
          <h2><i :class="lesson.icon"></i> {{ lesson.topic }}</h2>
          <p><i class="fa-solid fa-location-dot"></i> {{ lesson.location }}</p>
          <p><i class="fa-solid fa-sterling-sign"></i> {{ lesson.price }}</p>
          <p><i class="fa-solid fa-users"></i> Spaces: {{ lesson.space }}</p>
          <button :disabled="lesson.space === 0" @click="addToCart(lesson)">Add to Cart</button>
        </div>
      </div>
    </div>

    <!-- cart page with checkout -->
    <div v-else class="cart">
      <h2>Your Cart</h2>

      <div class="cart-item" v-for="item in cart" :key="item.lessonId">
        <span>{{ item.topic }} ({{ item.location }})</span>
        <span>{{ item.spaces }} x £{{ item.price }}</span>
        <button class="remove" @click="removeFromCart(item)">Remove</button>
      </div>

      <p class="total">Total: £{{ totalPrice }}</p>

      <h2>Checkout</h2>
      <input v-model="name" placeholder="Name" />
      <p class="error" v-if="name !== '' && !nameValid">Name must be letters only</p>
      <input v-model="phone" placeholder="Phone" />
      <p class="error" v-if="phone !== '' && !phoneValid">Phone must be numbers only</p>

      <button :disabled="!nameValid || !phoneValid" @click="checkout">Checkout</button>
    </div>
  </div>
</template>

<style>
body {
  margin: 0;
  font-family: Arial, sans-serif;
  background: #f3f4f6;
  color: #222;
}

.page {
  max-width: 1000px;
  margin: 0 auto;
  padding: 20px;
}

header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

h1 {
  margin: 0;
  font-size: 26px;
}

button {
  background: #2c7be5;
  color: white;
  border: none;
  padding: 10px 16px;
  border-radius: 5px;
  cursor: pointer;
  font-size: 15px;
}

button:disabled {
  background: #aaa;
  cursor: not-allowed;
}

.controls {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  margin-bottom: 20px;
}

input,
select {
  padding: 9px;
  border: 1px solid #ccc;
  border-radius: 5px;
  font-size: 15px;
}

.lessons {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
  gap: 16px;
}

.lesson {
  background: white;
  padding: 14px;
  border-radius: 8px;
}

.lesson img {
  width: 100%;
  border-radius: 5px;
}

.lesson h2 {
  font-size: 20px;
  margin: 10px 0;
}

.lesson p {
  margin: 6px 0;
}

.lesson button {
  width: 100%;
  margin-top: 8px;
}

.cart {
  background: white;
  padding: 20px;
  border-radius: 8px;
}

.cart input {
  display: block;
  width: 100%;
  box-sizing: border-box;
  margin-bottom: 10px;
}

.cart-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 0;
  border-bottom: 1px solid #ddd;
}

.remove {
  background: #d9534f;
}

.total {
  font-weight: bold;
}

.message {
  background: #d4edda;
  padding: 12px;
  border-radius: 5px;
}

.error {
  color: #d9534f;
  margin: 0 0 10px;
}
</style>
