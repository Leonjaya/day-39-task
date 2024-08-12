# day-39-task
// models/Mentor.js
const mongoose = require('mongoose');

const mentorSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  students: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Student' }]
});

module.exports = mongoose.model('Mentor', mentorSchema);

// models/Student.js
const mongoose = require('mongoose');

const studentSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  mentor: { type: mongoose.Schema.Types.ObjectId, ref: 'Mentor' }
});

module.exports = mongoose.model('Student', studentSchema);
const express = require('express');
const mongoose = require('mongoose');
const Mentor = require('./models/Mentor');
const Student = require('./models/Student');

const app = express();
app.use(express.json());

mongoose.connect('mongodb://localhost:27017/zenClass', {
  useNewUrlParser: true,
  useUnifiedTopology: true
}).then(() => console.log('Connected to MongoDB'))
  .catch(err => console.error('Could not connect to MongoDB', err));

app.post('/mentors', async (req, res) => {
  try {
    const { name, email } = req.body;
    const mentor = new Mentor({ name, email });
    await mentor.save();
    res.status(201).json(mentor);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});
app.post('/students', async (req, res) => {
  try {
    const { name, email } = req.body;
    const student = new Student({ name, email });
    await student.save();
    res.status(201).json(student);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});
app.post('/mentors/:mentorId/assign-students', async (req, res) => {
  try {
    const { mentorId } = req.params;
    const { studentIds } = req.body;  // Array of student IDs

    const mentor = await Mentor.findById(mentorId);
    if (!mentor) return res.status(404).json({ error: 'Mentor not found' });

    await Student.updateMany(
      { _id: { $in: studentIds }, mentor: { $exists: false } },
      { mentor: mentorId }
    );

    mentor.students.push(...studentIds);
    await mentor.save();

    res.status(200).json({ message: 'Students assigned successfully' });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});
app.get('/mentors/:mentorId/students', async (req, res) => {
  try {
    const { mentorId } = req.params;
    const mentor = await Mentor.findById(mentorId).populate('students');
    if (!mentor) return res.status(404).json({ error: 'Mentor not found' });

    res.status(200).json(mentor.students);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});
app.post('/students/:studentId/assign-mentor', async (req, res) => {
  try {
    const { studentId } = req.params;
    const { mentorId } = req.body;

    const student = await Student.findById(studentId);
    if (!student) return res.status(404).json({ error: 'Student not found' });

    const previousMentor = await Mentor.findById(student.mentor);
    if (previousMentor) {
      previousMentor.students.pull(studentId);
      await previousMentor.save();
    }

    const newMentor = await Mentor.findById(mentorId);
    if (!newMentor) return res.status(404).json({ error: 'Mentor not found' });

    student.mentor = mentorId;
    await student.save();

    newMentor.students.push(studentId);
    await newMentor.save();

    res.status(200).json({ message: 'Mentor assigned successfully' });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});
app.get('/students/:studentId/previous-mentor', async (req, res) => {
  try {
    const { studentId } = req.params;

    const student = await Student.findById(studentId).populate('mentor');
    if (!student) return res.status(404).json({ error: 'Student not found' });

    res.status(200).json(student.mentor);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server is running on port ${PORT}`);
});
