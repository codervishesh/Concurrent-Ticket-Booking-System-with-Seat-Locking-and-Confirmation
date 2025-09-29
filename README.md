# Concurrent-Ticket-Booking-System-with-Seat-Locking-and-Confirmation
Create a Node.js and Express.js application that simulates a ticket booking system for events or movie theaters. Implement endpoints to view available seats, temporarily lock a seat for a user, and confirm the booking.
const express = require("express");
const app = express();
const port = 3000;

app.use(express.json());

// In-memory seat state:
// 'available': Seat is free.
// 'locked': Seat is temporarily reserved by a user.
// 'booked': Seat is permanently booked.
let seats = {
  "1": { status: "available", lockTime: null },
  "2": { status: "available", lockTime: null },
  "3": { status: "available", lockTime: null },
  "4": { status: "available", lockTime: null },
  "5": { status: "available", lockTime: null }
};

const LOCK_DURATION = 60000; // 60 seconds (1 minute)

// Middleware to clean up expired locks before handling a request
const cleanupExpiredLocks = (req, res, next) => {
  const now = Date.now();
  for (const id in seats) {
    const seat = seats[id];
    if (seat.status === "locked" && seat.lockTime && (now - seat.lockTime > LOCK_DURATION)) {
      seat.status = "available";
      seat.lockTime = null;
    }
  }
  next();
};

app.use(cleanupExpiredLocks);

// GET /seats - View available seats
app.get("/seats", (req, res) => {
  const displaySeats = {};
  for (const id in seats) {
    displaySeats[id] = { status: seats[id].status };
  }
  res.json(displaySeats);
});

// POST /lock/:id - Temporarily lock a seat
app.post("/lock/:id", (req, res) => {
  const seatId = req.params.id;
  const seat = seats[seatId];

  if (!seat) {
    return res.status(404).json({ message: "Seat not found" });
  }

  if (seat.status === "booked") {
    return res.status(409).json({ message: `Seat ${seatId} is already booked` });
  }

  if (seat.status === "locked") {
    return res.status(409).json({ message: `Seat ${seatId} is already locked. Try again later` });
  }

  // Lock the seat
  seat.status = "locked";
  seat.lockTime = Date.now();

  res.status(200).json({ message: `Seat ${seatId} locked successfully. Confirm within ${LOCK_DURATION / 1000} seconds.` });
});

// POST /confirm/:id - Confirm the booking for a locked seat
app.post("/confirm/:id", (req, res) => {
  const seatId = req.params.id;
  const seat = seats[seatId];

  if (!seat) {
    return res.status(404).json({ message: "Seat not found" });
  }

  // A seat can only be booked if its status is 'locked'
  if (seat.status !== "locked") {
    return res.status(400).json({ message: "Seat is not locked and cannot be booked" });
  }

  // The lock must be current (not expired)
  if (seat.lockTime && (Date.now() - seat.lockTime > LOCK_DURATION)) {
    // This case is largely handled by cleanupExpiredLocks, but good for explicit check
    seat.status = "available";
    seat.lockTime = null;
    return res.status(400).json({ message: "Seat lock has expired. Please lock the seat again" });
  }

  // Confirm the booking
  seat.status = "booked";
  seat.lockTime = null; // Clear lock time
  
  res.status(200).json({ message: `Seat ${seatId} booked successfully!` });
});

app.listen(port, () => {
  console.log(`Server running on http://localhost:${port}`);
});
