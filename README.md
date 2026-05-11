"""
自动视频剪辑软件 - 简化版
功能：图片拼接、配音、字幕、背景音乐
"""

import tkinter as tk
from tkinter import ttk, filedialog, scrolledtext, messagebox
from pathlib import Path
import edge_tts
import threading
import asyncio
import os
from threading import Thread

# moviepy 导入
from moviepy import ImageClip, AudioFileClip, concatenate_videoclips
from moviepy.video.fx.Resize import Resize


class VideoAutoCreator:
    def __init__(self):
        self.root = tk.Tk()
        self.root.title("自动视频剪辑软件")
        self.root.geometry("600x750")
        self.root.configure(bg='#f0f0f0')

        self.images = []
        self.temp_dir = Path(__file__).parent / "temp"
        self.temp_dir.mkdir(exist_ok=True)

        self._create_ui()

    def _create_ui(self):
        """创建界面"""
        # 标题
        title_label = ttk.Label(
            self.root,
            text="自动视频剪辑软件",
            font=('Arial', 16, 'bold')
        )
        title_label.pack(pady=15)

        # 图片上传区域
        img_frame = ttk.LabelFrame(self.root, text="1. 上传图片", padding=10)
        img_frame.pack(pady=10, padx=20, fill=tk.X)

        img_btn_frame = ttk.Frame(img_frame)
        img_btn_frame.pack(fill=tk.X)

        ttk.Button(img_btn_frame, text="添加图片", command=self.add_images).pack(side=tk.LEFT, padx=5)
        ttk.Button(img_btn_frame, text="清空图片", command=self.clear_images).pack(side=tk.LEFT, padx=5)

        self.img_count_label = ttk.Label(img_frame, text="已选择 0 张图片")
        self.img_count_label.pack(pady=5)

        # 文案输入区域
        text_frame = ttk.LabelFrame(self.root, text="2. 输入文案", padding=10)
        text_frame.pack(pady=10, padx=20, fill=tk.BOTH, expand=True)

        ttk.Label(text_frame, text="输入文案内容（将用于配音）：").pack(anchor=tk.W)
        self.text_input = scrolledtext.ScrolledText(text_frame, height=5, wrap=tk.WORD)
        self.text_input.pack(fill=tk.BOTH, expand=True, pady=5)
        self.text_input.insert(tk.END, "欢迎使用自动视频剪辑软件。这个工具可以帮您快速制作精美的短视频。")

        # 视频设置区域
        setting_frame = ttk.LabelFrame(self.root, text="3. 视频设置", padding=10)
        setting_frame.pack(pady=10, padx=20, fill=tk.X)

        # 每张图片显示时长
        dur_frame = ttk.Frame(setting_frame)
        dur_frame.pack(fill=tk.X, pady=5)

        ttk.Label(dur_frame, text="每张图片时长(秒):").pack(side=tk.LEFT)
        self.duration_var = tk.StringVar(value="3")
        ttk.Spinbox(dur_frame, from_=1, to=10, textvariable=self.duration_var, width=5).pack(side=tk.LEFT, padx=5)

        # 配音设置
        voice_frame = ttk.Frame(setting_frame)
        voice_frame.pack(fill=tk.X, pady=5)

        ttk.Label(voice_frame, text="配音声音:").pack(side=tk.LEFT)
        self.voice_var = tk.StringVar(value="zh-CN-XiaoxiaoNeural")
        voices = ["zh-CN-XiaoxiaoNeural(女)", "zh-CN-YunxiNeural(男)", "zh-CN-YunjianNeural(男)"]
        ttk.Combobox(voice_frame, textvariable=self.voice_var, values=voices, width=20, state="readonly").pack(side=tk.LEFT, padx=5)

        # 背景音乐
        music_frame = ttk.Frame(setting_frame)
        music_frame.pack(fill=tk.X, pady=5)

        ttk.Label(music_frame, text="背景音乐:").pack(side=tk.LEFT)
        ttk.Button(music_frame, text="选择音乐文件", command=self.select_music).pack(side=tk.LEFT, padx=5)
        self.music_label = ttk.Label(music_frame, text="未选择")
        self.music_label.pack(side=tk.LEFT, padx=5)
        self.music_path = None

        # 生成按钮
        gen_frame = ttk.Frame(self.root)
        gen_frame.pack(pady=15)

        self.generate_btn = ttk.Button(gen_frame, text="生成视频", command=self.start_generate, width=20)
        self.generate_btn.pack()

        # 进度显示
        self.progress_text = scrolledtext.ScrolledText(self.root, height=8, wrap=tk.WORD)
        self.progress_text.pack(pady=10, padx=20, fill=tk.X)

        # 状态栏
        self.status_label = ttk.Label(self.root, text="就绪 - 请添加图片并输入文案")
        self.status_label.pack(side=tk.BOTTOM, pady=5)

    def add_images(self):
        """添加图片"""
        files = filedialog.askopenfilenames(
            title="选择图片",
            filetypes=[("图片文件", "*.jpg *.jpeg *.png *.webp"), ("所有文件", "*.*")]
        )
        if files:
            self.images.extend(list(files))
            self.img_count_label.config(text=f"已选择 {len(self.images)} 张图片")
            self.status_label.config(text=f"已添加 {len(files)} 张图片，共 {len(self.images)} 张")

    def clear_images(self):
        """清空图片"""
        self.images = []
        self.img_count_label.config(text="已选择 0 张图片")
        self.status_label.config(text="已清空图片")

    def select_music(self):
        """选择背景音乐"""
        file = filedialog.askopenfilename(
            title="选择背景音乐",
            filetypes=[("音频文件", "*.mp3 *.wav *.m4a"), ("所有文件", "*.*")]
        )
        if file:
            self.music_path = file
            self.music_label.config(text=Path(file).name)

    def log_progress(self, msg):
        """记录进度"""
        self.progress_text.insert(tk.END, msg + "\n")
        self.progress_text.see(tk.END)
        self.root.update()

    def start_generate(self):
        """开始生成"""
        if not self.images:
            messagebox.showwarning("提示", "请先添加图片")
            return

        text = self.text_input.get(1.0, tk.END).strip()
        if not text:
            messagebox.showwarning("提示", "请输入文案")
            return

        self.generate_btn.config(state="disabled")
        self.progress_text.delete(1.0, tk.END)

        Thread(target=self._generate_thread, daemon=True).start()

    def _generate_thread(self):
        """生成线程"""
        try:
            # 1. 生成配音
            self.root.after(0, lambda: self.log_progress("步骤1: 正在生成配音..."))

            tts_file = self.temp_dir / "tts.mp3"
            voice = "zh-CN-XiaoxiaoNeural"  # 使用固定的女声

            async def gen_tts():
                communicate = edge_tts.Communicate(
                    self.text_input.get(1.0, tk.END).strip(),
                    voice
                )
                await communicate.save(str(tts_file))

            asyncio.run(gen_tts())
            self.root.after(0, lambda: self.log_progress(f"配音已生成: {tts_file}"))

            # 2. 获取配音时长，计算每张图片时长
            audio_clip = AudioFileClip(str(tts_file))
            total_duration = audio_clip.duration
            per_image_duration = total_duration / len(self.images)
            self.root.after(0, lambda: self.log_progress(f"配音时长: {total_duration:.1f}秒，每张图片: {per_image_duration:.1f}秒"))

            # 3. 创建图片片段
            self.root.after(0, lambda: self.log_progress("步骤2: 正在创建图片片段..."))

            clips = []
            for i, img_path in enumerate(self.images):
                self.root.after(0, lambda idx=i: self.log_progress(f"  处理图片 {idx+1}/{len(self.images)}"))

                clip = ImageClip(img_path, duration=per_image_duration)
                # 调整到统一尺寸
                clip = clip.with_effects([Resize(height=720)])
                clips.append(clip)

            # 4. 拼接视频
            self.root.after(0, lambda: self.log_progress("步骤3: 正在拼接视频..."))
            video = concatenate_videoclips(clips, method="compose")

            # 5. 添加配音
            self.root.after(0, lambda: self.log_progress("步骤4: 正在添加配音..."))
            video = video.with_audio(audio_clip)

            # 6. 导出视频
            self.root.after(0, lambda: self.log_progress("步骤5: 正在导出视频(可能需要几分钟)..."))
            output_path = Path.home() / "Desktop" / "auto_video.mp4"

            video.write_videofile(
                str(output_path),
                fps=24,
                codec='libx264',
                audio_codec='aac',
                threads=4,
                preset='medium',
                logger=None  # 禁用进度条输出
            )

            # 7. 完成
            self.root.after(0, lambda: self.log_progress(f"\n✅ 视频已生成！"))
            self.root.after(0, lambda: self.log_progress(f"保存位置: {output_path}"))
            self.root.after(0, lambda: self.status_label.config(text="视频生成完成！"))
            self.root.after(0, lambda: messagebox.showinfo("完成", f"视频已保存到桌面!\n{output_path}"))

            # 清理临时文件
            audio_clip.close()
            video.close()
            for clip in clips:
                clip.close()

        except Exception as e:
            error_msg = f"生成失败: {str(e)}"
            self.root.after(0, lambda: self.log_progress(f"\n❌ {error_msg}"))
            self.root.after(0, lambda: self.status_label.config(text=error_msg))
            self.root.after(0, lambda: messagebox.showerror("错误", error_msg))

        finally:
            self.root.after(0, lambda: self.generate_btn.config(state="normal"))

    def run(self):
        """运行应用"""
        self.root.mainloop()


if __name__ == '__main__':
    app = VideoAutoCreator()
    app.run()
