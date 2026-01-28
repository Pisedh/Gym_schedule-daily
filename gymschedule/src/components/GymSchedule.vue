<template>
  <div class="schedule">
    <h1>Weekly Gym Schedule</h1>
    <table>
      <thead>
        <tr>
          <th>Time</th>
          <th>Monday</th>
          <th>Tuesday</th>
          <th>Wednesday</th>
          <th>Thursday</th>
          <th>Friday</th>
          <th>Saturday</th>
          <th>Sunday</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="item in schedule" :key="item.time">
          <td>{{ item.time }}</td>
          <td>{{ item.monday }}</td>
          <td>{{ item.tuesday }}</td>
          <td>{{ item.wednesday }}</td>
          <td>{{ item.thursday }}</td>
          <td>{{ item.friday }}</td>
          <td>{{ item.saturday }}</td>
          <td>{{ item.sunday }}</td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script>
export default {
  name: 'GymSchedule',
  data() {
    return {
      schedule: [
        { time: '6:00 AM - 8:00 AM', monday: 'Yoga', tuesday: '', wednesday: 'Yoga', thursday: '', friday: 'Yoga', saturday: 'Cardio', sunday: '' },
        { time: '8:00 AM - 10:00 AM', monday: 'Cardio', tuesday: 'Weights', wednesday: 'Cardio', thursday: 'Weights', friday: 'Cardio', saturday: '', sunday: 'Weights' },
        { time: '4:00 PM - 6:00 PM', monday: 'Weights', tuesday: 'Cardio', wednesday: 'Weights', thursday: 'Cardio', friday: 'Weights', saturday: 'Yoga', sunday: '' },
        { time: '6:00 PM - 8:00 PM', monday: '', tuesday: 'Yoga', wednesday: '', thursday: 'Yoga', wednesday: '', thursday: 'Yoga', friday: '', saturday: '', sunday: 'Cardio' }
      ]
    }
  }
}
</script>

<style scoped>
.schedule {
  font-family: sans-serif;
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
}
table {
  border-collapse: collapse;
  margin: 20px 0;
  font-size: 0.9em;
  min-width: 400px;
  box-shadow: 0 0 20px rgba(0, 0, 0, 0.15);
}
thead tr {
  background-color: #009879;
  color: #ffffff;
  text-align: left;
}
th, td {
  padding: 12px 15px;
}
tbody tr {
  border-bottom: 1px solid #dddddd;
}
tbody tr:nth-of-type(even) {
  background-color: #f3f3f3;
}
tbody tr:last-of-type {
  border-bottom: 2px solid #009879;
}
</style>
