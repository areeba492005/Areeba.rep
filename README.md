import cv2
import mediapipe as mp
import numpy as np
import time
import random
import colorsys

# MediaPipe Setup
mp_hands = mp.solutions.hands
mp_draw = mp.solutions.drawing_utils

hand_detector = mp_hands.Hands(
    max_num_hands=2,
    model_complexity=1,
    min_detection_confidence=0.65,
    min_tracking_confidence=0.65
)

# New Theme Colors
LANDMARK_COLOR = (0, 255, 255)     # Yellow
CONNECTION_COLOR = (255, 140, 0)   # Orange

# Particle Storage
neon_particles = []


def rainbow_to_bgr(hue, saturation, lightness):
    r, g, b = colorsys.hls_to_rgb(hue / 360.0, lightness, saturation)
    return (int(b * 255), int(g * 255), int(r * 255))


def generate_particles(x_pos, y_pos, particle_color):
    neon_particles.append({
        "x": x_pos,
        "y": y_pos,
        "vx": random.uniform(-4, 4),
        "vy": random.uniform(-4, 4),
        "life": 0.8,
        "size": random.randint(2, 5),
        "color": particle_color
    })


def animate_particles(frame):
    for particle in reversed(neon_particles):

        particle["x"] += particle["vx"]
        particle["y"] += particle["vy"]

        particle["life"] -= 0.025

        if particle["life"] <= 0:
            neon_particles.remove(particle)
            continue

        cv2.circle(
            frame,
            (int(particle["x"]), int(particle["y"])),
            particle["size"],
            particle["color"],
            -1
        )


# Webcam Setup
camera = cv2.VideoCapture(0)

camera.set(cv2.CAP_PROP_FRAME_WIDTH, 1280)
camera.set(cv2.CAP_PROP_FRAME_HEIGHT, 720)

start_clock = time.time()

while camera.isOpened():

    ret, frame = camera.read()

    if not ret:
        break

    frame = cv2.flip(frame, 1)

    frame_height, frame_width, _ = frame.shape

    glow_layer = frame.copy()

    rgb_frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)

    result = hand_detector.process(rgb_frame)

    if result.multi_hand_landmarks:

        for hand in result.multi_hand_landmarks:

            # Thick Glow Layer
            mp_draw.draw_landmarks(
                glow_layer,
                hand,
                mp_hands.HAND_CONNECTIONS,
                mp_draw.DrawingSpec(
                    color=LANDMARK_COLOR,
                    thickness=9,
                    circle_radius=7
                ),
                mp_draw.DrawingSpec(
                    color=CONNECTION_COLOR,
                    thickness=12
                )
            )

            # Main Skeleton
            mp_draw.draw_landmarks(
                frame,
                hand,
                mp_hands.HAND_CONNECTIONS,
                mp_draw.DrawingSpec(
                    color=LANDMARK_COLOR,
                    thickness=3,
                    circle_radius=4
                ),
                mp_draw.DrawingSpec(
                    color=CONNECTION_COLOR,
                    thickness=3
                )
            )

        # Dual-Hand Neon Connections
        if len(result.multi_hand_landmarks) == 2:

            first_hand = result.multi_hand_landmarks[0].landmark
            second_hand = result.multi_hand_landmarks[1].landmark

            elapsed_ms = (time.time() - start_clock) * 1000

            for idx in range(4, len(first_hand), 2):

                point1 = (
                    int(first_hand[idx].x * frame_width),
                    int(first_hand[idx].y * frame_height)
                )

                point2 = (
                    int(second_hand[idx].x * frame_width),
                    int(second_hand[idx].y * frame_height)
                )

                dynamic_hue = (
                    idx * 50 +
                    elapsed_ms * 0.04
                ) % 360

                neon_color = rainbow_to_bgr(
                    dynamic_hue,
                    1.0,
                    0.55
                )

                cv2.line(
                    glow_layer,
                    point1,
                    point2,
                    neon_color,
                    8
                )

                cv2.line(
                    frame,
                    point1,
                    point2,
                    neon_color,
                    3
                )

            # Generate Particle Effects
            for finger_tip in range(4, 21):

                for hand_number in range(
                        len(result.multi_hand_landmarks)):

                    tip = result.multi_hand_landmarks[
                        hand_number
                    ].landmark[finger_tip]

                    particle_hue = (
                        finger_tip * 30 +
                        hand_number * 120
                    ) % 360

                    generate_particles(
                        tip.x * frame_width,
                        tip.y * frame_height,
                        rainbow_to_bgr(
                            particle_hue,
                            1.0,
                            0.60
                        )
                    )

    # Enhanced Glow Blend
    output_frame = cv2.addWeighted(
        glow_layer,
        0.5,
        frame,
        0.5,
        0
    )

    animate_particles(output_frame)

    cv2.imshow(
        "Cyber Pulse Hand Tracker",
        output_frame
    )

    if cv2.waitKey(5) & 0xFF == 27:
        break

camera.release()
cv2.destroyAllWindows()
