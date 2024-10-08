import tkinter as tk
from tkinter import LabelFrame, PhotoImage
import cv2
from PIL import Image, ImageTk
import os
import datetime
import telegram
import asyncio
import tkintermapview
from ultralytics import YOLO
from djitellopy import Tello

# YOLO 모델 로드
model = YOLO('best_final.pt')

# 텔레그램 봇 설정
TELEGRAM_TOKEN = '7527808615:AAFImz7JnYsFYTt1i1Mxl2Uh5A3UkyLD-5E'  # 봇 토큰을 입력하세요
CHAT_ID = '6895411397'  # 올바른 채팅 ID를 입력하세요

# 텔레그램 봇 객체 생성
bot = telegram.Bot(token=TELEGRAM_TOKEN)

# Tello 드론 객체 생성
tello = Tello()
tello.connect()  # 드론 연결
tello.streamon()  # 비디오 스트림 시작

current_frame = None  # 전역 변수로 설정하여 모든 함수에서 접근 가능하도록 함
captured_image_label = None  # 캡처된 이미지를 표시할 라벨


async def send_image_via_telegram(file_path, additional_image_path=None):
    """이미지와 메시지를 텔레그램으로 전송하는 함수"""
    try:
        with open(file_path, 'rb') as f:
            await bot.send_photo(chat_id=CHAT_ID, photo=f, caption="쓰레기를 발견했습니다.")
        print(f"이미지가 텔레그램으로 전송되었습니다: {file_path}")

        # 추가 이미지 전송
        if additional_image_path:
            with open(additional_image_path, 'rb') as f:
                await bot.send_photo(chat_id=CHAT_ID, photo=f, caption="현재위치"
                                                                       "위도:35.157"
                                                                       "경도:129.163")
            print(f"추가 이미지가 텔레그램으로 전송되었습니다: {additional_image_path}")

    except Exception as e:
        print(f"텔레그램 전송 중 오류가 발생했습니다: {e}")


def get_box_color(class_name):
    colors = {
        'Aluminum-can': (0, 255, 0),  # 녹색
        'Plastic-bottle': (255, 0, 0),  # 파란색
        'Buoy': (0, 0, 255)  # 빨간색
    }
    return colors.get(class_name, (255, 255, 0))  # 기본값: 노란색


def video_stream():
    global current_frame

    # Tello 드론에서 프레임을 가져옴
    frame = tello.get_frame_read().frame  # 드론 비디오 스트림 프레임 가져오기
    current_frame = frame.copy()  # 현재 프레임 저장

    results = model(frame, conf=0.65)
    detected_objects = []  # 인식된 객체 목록 초기화

    for result in results:
        for box in result.boxes:
            if box.conf.item() > 0.65:
                x1, y1, x2, y2 = map(int, box.xyxy[0])  # 바운딩 박스 좌표
                class_id = int(box.cls.item())  # 클래스 인덱스
                class_name = result.names[class_id]  # 클래스 이름
                confidence = box.conf.item()  # 신뢰도 (정확도)
                label = f"{class_name} {confidence:.2f}"

                # 객체 정보를 리스트에 저장
                detected_objects.append((class_name, confidence))

                box_color = get_box_color(class_name)  # 바운딩 박스 색상 결정
                cv2.rectangle(frame, (x1, y1), (x2, y2), box_color, 2)
                cv2.putText(frame, label, (x1, y1 - 10), cv2.FONT_HERSHEY_SIMPLEX, 0.5, box_color, 2)

        # 인식된 객체 목록, 개체 수, 정확도 표시 (각각 한 줄로)
        if detected_objects:
            object_names = ", ".join([obj[0] for obj in detected_objects])
            count = len(detected_objects)
            average_confidence = sum(obj[1] for obj in detected_objects) / count

            text_lines = [
                f"Objects: {object_names}",
                f"Count: {count}",
                f"Confidence: {average_confidence:.2f}"
            ]

            colors = [(255, 255, 255), (255, 0, 0), (0, 255, 0)]  # 각각의 색깔

            start_y = 30  # 첫 번째 텍스트의 시작 위치 (y 좌표)
            for idx, line in enumerate(text_lines):
                cv2.putText(frame, line, (10, start_y), cv2.FONT_HERSHEY_SIMPLEX, 0.6, colors[idx], 2)
                start_y += 30  # 줄 띄우기

    current_frame = frame.copy() # 현재 프레임을 바운딩 박스가 그려진 상태로 저장

    frame_rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
    frame_resized = cv2.resize(frame_rgb, (800, 800))
    image = Image.fromarray(frame_resized)
    image_tk = ImageTk.PhotoImage(image)

    video_label.config(image=image_tk)
    video_label.image = image_tk

    window.after(10, video_stream)


def save_detected_image(frame, label):
    """이미지를 바탕화면에 저장하고 텔레그램으로 전송"""
    global captured_image_label

    # 바탕화면에 'Captured_Images' 폴더 만들기
    desktop_path = os.path.join(os.path.expanduser("~"), "Desktop")
    folder_name = "Captured_Images"
    folder_path = os.path.join(desktop_path, folder_name)

    if not os.path.exists(folder_path):
        os.makedirs(folder_path)

    # 현재 시간 기반 파일 이름 생성
    timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
    file_path = os.path.join(folder_path, f"capture_{label}_{timestamp}.png")

    # 이미지 저장
    frame_rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
    image = Image.fromarray(frame_rgb)
    image.save(file_path)

    print(f"이미지가 저장되었습니다: {file_path}")

    # 비동기식으로 텔레그램으로 이미지 전송
    asyncio.run(send_image_via_telegram(file_path, additional_image_path='images/map.png'))

    # 캡처된 이미지를 오른쪽 하단에 표시
    captured_image = ImageTk.PhotoImage(image.resize((400, 300)))
    captured_image_label.config(image=captured_image)
    captured_image_label.image = captured_image


def capture_webcam_image():
    global current_frame
    if current_frame is None:
        print("저장할 프레임이 없습니다.")
        return
    save_detected_image(current_frame, "drone_capture")


def execute_drone_command(command):
    """드론 명령을 실행하는 함수 (디버그용 메시지 출력)"""
    print(f"드론 명령 실행: {command}")


def show_interface():
    global video_label, window, captured_image_label

    window = tk.Tk()
    window.title("AI Water Trash Detection Drone")
    window.geometry('1200x800+100+100')
    window.resizable(True, True)
    window.option_add('*Font', '맑은고딕 15')
    dark_bg_color = "#ffffff"
    window.configure(bg="#D9D9D9")

    # 왼쪽 프레임: 드론 비디오 스트리밍
    left_frame = tk.Frame(window, bg="#D9D9D9")
    left_frame.grid(row=0, column=0, padx=0, pady=10, sticky="nsew")

    video_label = tk.Label(left_frame, bg="#D9D9D9")
    video_label.grid(row=0, column=0, padx=0, pady=10)  # 간격 없이 배치

    # 오른쪽 프레임: 지도 및 캡처된 이미지
    right_frame = tk.Frame(window, bg="#D9D9D9")
    right_frame.grid(row=0, column=1, padx=0, pady=10, sticky="nsew")

    # 캡처된 이미지 라벨 (오른쪽 하단)
    captured_image_label = tk.Label(right_frame, bg="#D9D9D9")
    captured_image_label.grid(row=1, column=0, padx=0, pady=10)

    button_frame = tk.Frame(window, bg="#D9D9D9")
    button_frame.grid(row=1, column=0, columnspan=2, padx=10, pady=10, sticky="s")

    # 지도 생성 (오른쪽 상단)
    map_label = LabelFrame(right_frame, text="Map", padx=0, pady=10)
    map_label.grid(row=0, column=0)

    map_widget = tkintermapview.TkinterMapView(map_label, width=400, height=300, corner_radius=0)
    map_widget.set_position(35.15703823782956, 129.163116736629)
    map_widget.set_zoom(15)
    map_widget.grid(row=0, column=0)

    # 이미지 로드 및 버튼 설정
    left_image = Image.open(r"images\left_icon.png").resize((55, 55))
    right_image = Image.open(r"images\right_icon.png").resize(
        (55, 55))
    fwd_image = Image.open(r"images\fwd_icon.png").resize((55, 55))
    bwd_image = Image.open(r"images\bwd_icon.png").resize((55, 55))
    camera_image = Image.open(r"images\capture_icon.png").resize(
        (55, 55))
    takeoff_image = Image.open(r"images\takeoff_icon.png").resize(
        (55, 55))
    landing_image = Image.open(r"images\landing_icon.png").resize(
        (55, 55))
    up_image = Image.open(r"images\up_icon.png").resize((55, 55))
    down_image = Image.open(r"images\down_icon.png").resize((55, 55))

    # 크기 조정된 이미지를 PhotoImage로 변환
    left_image = ImageTk.PhotoImage(left_image)
    right_image = ImageTk.PhotoImage(right_image)
    fwd_image = ImageTk.PhotoImage(fwd_image)
    bwd_image = ImageTk.PhotoImage(bwd_image)
    camera_image = ImageTk.PhotoImage(camera_image)
    takeoff_image = ImageTk.PhotoImage(takeoff_image)
    landing_image = ImageTk.PhotoImage(landing_image)
    up_image = ImageTk.PhotoImage(up_image)
    down_image = ImageTk.PhotoImage(down_image)

    tk.Button(button_frame, image=left_image, borderwidth=0, highlightthickness=0, relief="flat",
              bg="#D9D9D9", command=lambda: execute_drone_command(lambda: print("왼쪽으로 이동"))).grid(row=1, column=1,
                                                                                                  padx=5, pady=5)
    tk.Button(button_frame, image=right_image, borderwidth=0, highlightthickness=0, relief="flat",
              bg="#D9D9D9", command=lambda: execute_drone_command(lambda: print("오른쪽으로 이동"))).grid(row=1, column=3,
                                                                                                   padx=5, pady=5)
    tk.Button(button_frame, image=fwd_image, borderwidth=0, highlightthickness=0, relief="flat",
              bg="#D9D9D9", command=lambda: execute_drone_command(lambda: print("앞으로 이동"))).grid(row=0, column=2,
                                                                                                 padx=5, pady=5)

    # 카메라 버튼을 "앞으로 이동"과 "뒤로 이동" 버튼 사이에 추가
    tk.Button(button_frame, image=camera_image, borderwidth=0, highlightthickness=0, relief="flat",
              bg="#D9D9D9", command=capture_webcam_image).grid(row=1, column=2, padx=5, pady=5)

    tk.Button(button_frame, image=bwd_image, borderwidth=0, highlightthickness=0, relief="flat",
              bg="#D9D9D9", command=lambda: execute_drone_command(lambda: print("뒤로 이동"))).grid(row=2, column=2, padx=5,
                                                                                                pady=5)

    tk.Button(button_frame, image=takeoff_image, borderwidth=0, highlightcolor="#D9D9D9", highlightthickness=0,
              bg="#D9D9D9", relief="flat", command=lambda: execute_drone_command(lambda: print("이륙"))).grid(row=0,
                                                                                                            column=0,
                                                                                                            padx=5,
                                                                                                            pady=5)
    tk.Button(button_frame, image=landing_image, borderwidth=0, highlightcolor="#D9D9D9", highlightthickness=0,
              bg="#D9D9D9", relief="flat", command=lambda: execute_drone_command(lambda: print("착륙"))).grid(row=2,
                                                                                                            column=0,
                                                                                                            padx=5,
                                                                                                            pady=5)

    tk.Button(button_frame, image=up_image, borderwidth=0, highlightcolor="#D9D9D9", highlightthickness=0,
              bg="#D9D9D9", relief="flat", command=lambda: execute_drone_command(lambda: print("위로 이동"))).grid(row=0,
                                                                                                               column=4,
                                                                                                               padx=5,
                                                                                                               pady=5)
    tk.Button(button_frame, image=down_image, borderwidth=0, highlightcolor="#D9D9D9", highlightthickness=0,
              bg="#D9D9D9", relief="flat", command=lambda: execute_drone_command(lambda: print("아래로 이동"))).grid(row=2,
                                                                                                                column=4,

                                                                                                                pady=10)

    # 마커 표시를 위한 코드
    try:
        map_widget.set_marker(35.15858248406396, 129.16289572564162, text="알루미늄 캔")
                              #marker_color_circle="#ACA8F0", marker_color_outside="#4B3FF0")
        # map_widget.set_marker(35.15858248406396, 129.16289572564162)
        map_widget.set_marker(35.15745301596239, 129.15852061187078, text="플라스틱",
                              marker_color_circle="#1E259A", marker_color_outside="#422EC5") #blue
        map_widget.set_marker(35.157549013185836, 129.16223193865488, text="부표",
                              marker_color_circle="#07B22C", marker_color_outside="#10E062") #green
        map_widget.set_marker(35.15664394179215, 129.16547782028965, text="스티로폼",
                              marker_color_circle="#B99301", marker_color_outside="#EDE903") #yello
    except Exception as e:
        print("마커 표시 중 오류 발생:", e)


    window.grid_rowconfigure(0, weight=1)
    window.grid_rowconfigure(1, weight=0)
    window.grid_columnconfigure(0, weight=1)

    window.after(0, video_stream)  # 비디오 스트리밍 함수 호출
    window.mainloop()


show_interface()
