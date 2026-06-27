import cv2
import numpy as np
import time
import random
import colorsys

LANDMARK_COLOR = (0, 255, 255)     # Yellow
CONNECTION_COLOR = (255, 140, 0)   # Orange
neon_particles = []

def rainbow_to_bgr(hue, saturation, lightness):
    r, g, b = colorsys.hls_to_rgb(hue / 360.0, lightness, saturation)
    return (int(b * 255), int(g * 255), int(r * 255))

def generate_particles(x_pos, y_pos, particle_color):
    neon_particles.append({
        "x": x_pos, "y": y_pos,
        "vx": random.uniform(-4, 4), "vy": random.uniform(-4, 4),
        "life": 0.8, "size": random.randint(2, 5), "color": particle_color
    })

def animate_particles(frame):
    for particle in reversed(neon_particles):
        particle["x"] += particle["vx"]
        particle["y"] += particle["vy"]
        particle["life"] -= 0.025
        if particle["life"] <= 0:
            neon_particles.remove(particle)
            continue
        cv2.circle(frame, (int(particle["x"]), int(particle["y"])), particle["size"], particle["color"], -1)

camera = cv2.VideoCapture(0)
cv2.namedWindow("Cyber Pulse Simulator", cv2.WINDOW_AUTOSIZE)
start_clock = time.time()

print("Camera loading... Screen par window check karein. Exit karne ke liye ESC dabayein.")

while camera.isOpened():
    ret, frame = camera.read()
    if not ret:
        print("Webcam feed nahi mil rahi.")
        break

    frame = cv2.flip(frame, 1)
    frame_height, frame_width, _ = frame.shape
    glow_layer = frame.copy()
    
    elapsed_ms = (time.time() - start_clock) * 1000
    
    cx, cy = frame_width // 2, frame_height // 2
    
    fake_landmarks = [
        (cx, cy), (cx-50, cy+50), (cx+50, cy+50), 
        (cx-100, cy-100), (cx-50, cy-150), (cx, cy-170), (cx+50, cy-150), (cx+100, cy-100)
    ]
    
    for i, pt in enumerate(fake_landmarks):
        dynamic_hue = (i * 45 + elapsed_ms * 0.04) % 360
        neon_color = rainbow_to_bgr(dynamic_hue, 1.0, 0.55)
        
        cv2.circle(glow_layer, pt, 15, LANDMARK_COLOR, -1)
        cv2.circle(frame, pt, 6, (255, 255, 255), -1)
        
        if i > 0:
            cv2.line(glow_layer, fake_landmarks[i-1], pt, neon_color, 10)
            cv2.line(frame, fake_landmarks[i-1], pt, neon_color, 3)
            
        
        if i >= 3:
            generate_particles(pt[0], pt[1], neon_color)

    
    output_frame = cv2.addWeighted(glow_layer, 0.5, frame, 0.5, 0)
    animate_particles(output_frame)

    
    cv2.putText(output_frame, "Cyber Pulse: OpenCV Active", (30, 50), 
                cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 255, 0), 2)

    cv2.imshow("Cyber Pulse Simulator", output_frame)
    cv2.setWindowProperty("Cyber Pulse Simulator", cv2.WND_PROP_TOPMOST, 1)

    if cv2.waitKey(1) & 0xFF == 27:  # Press ESC to exit
        break
camera.release()
cv2.destroyAllWindows()
for i in range(5): cv2.waitKey(1)
print("Window closed safely.")
