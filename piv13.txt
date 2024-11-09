import os
import pandas as pd
import requests
import tkinter as tk
from tkinter import *
from tkinter import filedialog, messagebox, colorchooser, ttk
from PIL import Image, ImageTk, ImageDraw, ImageFont, ImageFilter
from io import BytesIO
from tkinter.ttk import Progressbar
from tkinter import Tk, Canvas, PhotoImage
import schedule
import time  # Para programar tareas
layer_app = None


class LayerManager:
    # Clase LayerManager vacía para referencia
    pass

class ImagePostApp:
    def load_background(self):
        # Lógica para cargar la imagen de fondo
        self.bg_image_path = filedialog.askopenfilename()
        if self.bg_image_path:
            try:
                background_image = Image.open(self.bg_image_path)
                self.bg_image_tk = ImageTk.PhotoImage(background_image)
                self.canvas.create_image(0, 0, image=self.bg_image_tk, anchor='nw')
                self.canvas.config(scrollregion=(0, 0, background_image.width, background_image.height))
                # Actualiza `self.image_data` si es necesario
                self.preview_image()  # Solo llama a preview_image si `self.image_data` está listo
            except Exception as e:
                messagebox.showerror("Error", f"No se pudo cargar la imagen de fondo: {str(e)}")
        else:
            messagebox.showwarning("Advertencia", "No se seleccionó ninguna imagen de fondo.")

        file_path = filedialog.askopenfilename(filetypes=[("Archivos de Imagen", "*.jpg;*.png;*.jpeg")])
        if file_path:
            self.bg_image_path = file_path
            self.bg_entry.delete(0, END)
            self.bg_entry.insert(0, file_path)
            self.preview_image()  # Para actualizar la vista previa

    def __init__(self, root):
        self.root = root
        self.root.title("Generador de Publicaciones")
        self.image_data = []
        self.current_index = 0
        self.bg_image_path = None
        self.font_path = "arial.ttf"
        self.output_dir = os.getcwd()
        self.zoom_level = 1.0
        self.create_widgets()
        self.title_position = (50, 50)
        self.price_position = (50, 100)
        self.root.grid_rowconfigure(3, weight=1)
        self.root.grid_columnconfigure(0, weight=1)
       # Botón para abrir el gestor de capas 
        self.layer_button = tk.Button(root, text="Editar Capas", command=self.open_layer_manager)
        self.layer_button.grid(row=0, column=0) # Usando grid en lugar de pack

        # Crear una instancia de LayerManager y LayerControlApp
        self.layer_manager = LayerManager()
        self.layer_app = None  # Definimos layer_app aquí para que sea un atributo de la instancia

    def open_layer_manager(self):
        # Crea una nueva ventana de Tkinter para `LayerControlApp`
        layer_manager_window = tk.Toplevel(self.root)
        layer_manager = LayerManager()
        layer_app = LayerControlApp(layer_manager_window, layer_manager)

    def close_layer_manager(self):
        # Cierra la ventana y elimina la referencia
        if self.layer_app is not None:
            self.layer_app.root.destroy()
            self.layer_app = None
        

    def create_widgets(self):
        # Crear el Canvas dentro del frame de imagenes
        self.image_frame = Frame(self.root, width=800, height=600)
        self.image_frame.grid(row=3, column=0, columnspan=3, sticky='nswe')
        
        self.canvas = Canvas(self.image_frame, width=800, height=600)
        self.canvas.pack(side=LEFT, fill=BOTH, expand=True)

        # Intentar cargar la imagen de fondo solo si existe la ruta
        if self.bg_image_path:
            try:
                # Abrir la imagen y convertirla para usar en tkinter
                background_image = Image.open(self.bg_image_path)
                self.bg_image_tk = ImageTk.PhotoImage(background_image)
                # Colocar la imagen en el Canvas
                self.canvas.create_image(0, 0, image=self.bg_image_tk, anchor='nw')
                # Configurar el scroll del canvas según el tamaño de la imagen de fondo
                self.canvas.config(scrollregion=(0, 0, background_image.width, background_image.height))
            except Exception as e:
                messagebox.showerror("Error", f"No se pudo cargar la imagen de fondo: {str(e)}")
        else:
            # Aviso al usuario de que no se ha cargado una imagen
            messagebox.showwarning("Advertencia", "No se ha cargado una imagen de fondo.")

        # Configurar el scroll del Canvas
        self.scroll_x = Scrollbar(self.image_frame, orient=HORIZONTAL, command=self.canvas.xview)
        self.scroll_x.pack(side=BOTTOM, fill=X)
        self.scroll_y = Scrollbar(self.image_frame, orient=VERTICAL, command=self.canvas.yview)
        self.scroll_y.pack(side=RIGHT, fill=Y)
        self.canvas.config(xscrollcommand=self.scroll_x.set, yscrollcommand=self.scroll_y.set)

        # Agregar otros widgets
        Label(self.root, text="Archivo de Excel:").grid(row=0, column=0)
        self.excel_entry = Entry(self.root, width=50)
        self.excel_entry.grid(row=0, column=1)
        Button(self.root, text="Cargar", command=self.load_excel).grid(row=0, column=2)

        Label(self.root, text="Plantilla de fondo:").grid(row=1, column=0)
        self.bg_entry = Entry(self.root, width=50)
        self.bg_entry.grid(row=1, column=1)
        Button(self.root, text="Cargar", command=self.load_background).grid(row=1, column=2)

        self.preview_button = Button(self.root, text="Previsualizar", command=self.preview_image)
        self.preview_button.grid(row=2, column=0, columnspan=3)

        # Barra de progreso
        self.progress = Progressbar(self.root, orient=HORIZONTAL, length=400, mode='determinate')
        self.progress.grid(row=4, column=0, columnspan=3)

        # Configuración de tamaño y color del título
        Label(self.root, text="Tamaño de Título:").grid(row=5, column=0)
        self.title_size = Scale(self.root, from_=10, to_=100, orient=HORIZONTAL)
        self.title_size.set(30)
        self.title_size.grid(row=5, column=1)

        Label(self.root, text="Tamaño de Precio:").grid(row=6, column=0)
        self.price_size = Scale(self.root, from_=10, to_=100, orient=HORIZONTAL)
        self.price_size.set(30)
        self.price_size.grid(row=6, column=1)

        Label(self.root, text="Color de Título:").grid(row=7, column=0)
        Button(self.root, text="Seleccionar Color", command=self.choose_title_color).grid(row=7, column=1)
        self.title_color_display = Label(self.root, text="", bg="black", width=10)
        self.title_color_display.grid(row=7, column=2)

        Label(self.root, text="Color de Precio:").grid(row=8, column=0)
        Button(self.root, text="Seleccionar Color", command=self.choose_price_color).grid(row=8, column=1)
        self.price_color_display = Label(self.root, text="", bg="black", width=10)
        self.price_color_display.grid(row=8, column=2)

        # Configuración del tamaño de imagen
        Label(self.root, text="Tamaño de Imagen:").grid(row=9, column=0)
        self.image_width = Scale(self.root, from_=100, to_=1080, orient=HORIZONTAL)
        self.image_width.set(1080)
        self.image_width.grid(row=9, column=1)
        self.image_height = Scale(self.root, from_=100, to_=1080, orient=HORIZONTAL)
        self.image_height.set(1080)
        self.image_height.grid(row=9, column=2)

        # Filtros
        Label(self.root, text="Filtros:").grid(row=10, column=0)
        self.filters = ttk.Combobox(self.root, values=["None", "BLUR", "CONTOUR", "DETAIL", "EDGE_ENHANCE", "SHARPEN"])
        self.filters.grid(row=10, column=1)

        # Botones de guardar y procesar todas las imágenes
        Button(self.root, text="Guardar", command=self.save_images).grid(row=11, column=0, columnspan=3)
        Button(self.root, text="Procesar Todas", command=self.process_all_images).grid(row=12, column=0, columnspan=3)

        # Configuración de carpeta de guardado
        Label(self.root, text="Carpeta de Guardado:").grid(row=13, column=0)
        self.output_entry = Entry(self.root, width=50)
        self.output_entry.grid(row=13, column=1)
        self.output_entry.insert(0, self.output_dir)
        Button(self.root, text="Seleccionar", command=self.select_output_dir).grid(row=13, column=2)

        # Eventos de arrastrar y soltar
        self.canvas.bind("<Button-1>", self.on_click)
        self.canvas.bind("<B1-Motion>", self.on_drag)


    def on_click(self, event):
        self.drag_data = {"x": event.x, "y": event.y, "item": None}
        if self.title_position[0] < event.x < self.title_position[0] + 100 and self.title_position[1] < event.y < self.title_position[1] + 30:
            self.drag_data["item"] = "title"
        elif self.price_position[0] < event.x < self.price_position[0] + 100 and self.price_position[1] < event.y < self.price_position[1] + 30:
            self.drag_data["item"] = "price"

    def on_drag(self, event):
        if self.drag_data["item"] == "title":
            self.title_position = (event.x, event.y)
        elif self.drag_data["item"] == "price":
            self.price_position = (event.x, event.y)
        self.preview_image()

    # Resto de funciones, incluyendo `load_excel`, `download_images`, `preview_image`, `create_preview`, `save_images`, y `process_all_images`.

    # Completar con el código de las funciones `load_excel`, `download_images`, `load_background`, `preview_image`, `create_preview`, `next_image`, `previous_image`, `save_images`, `process_all_images`.

# Resto de funciones, incluyendo `load_excel`, `download_images`, `preview_image`, y `create_preview`.

    
    
    def load_excel(self):
        file_path = filedialog.askopenfilename(filetypes=[("Excel files", "*.xlsx;*.xls")])
        if file_path:
            self.excel_entry.delete(0, END)
            self.excel_entry.insert(0, file_path)
            try:
                df = pd.read_excel(file_path)
                if 'URL' in df.columns and 'Título' in df.columns and 'Precio' in df.columns:
                    self.image_data = df[['URL', 'Título', 'Precio']].to_dict(orient='records')
                    self.current_index = 0
                    self.download_images()
                    self.preview_image()
                else:
                    messagebox.showerror("Error", "El archivo Excel debe contener las columnas 'URL', 'Título' y 'Precio'.")
            except Exception as e:
                messagebox.showerror("Error al cargar el archivo Excel", str(e))

    def download_images(self):
        self.progress['value'] = 0
        total = len(self.image_data)
        for index, data in enumerate(self.image_data):
            url = data['URL']
            try:
                if isinstance(url, str) and (url.startswith('http://') or url.startswith('https://')):
                    response = requests.get(url)
                    response.raise_for_status()  # Lanza un error si la solicitud falla
                    image = Image.open(BytesIO(response.content))
                    local_path = os.path.join(os.getcwd(), os.path.basename(url))
                    image.save(local_path)
                    data['local_path'] = local_path
                    print(f"Imagen descargada correctamente: {local_path}")
                else:
                    data['local_path'] = url  # Si la URL no es válida, simplemente usa la ruta local
                    print(f"Usando URL local: {url}")
            except Exception as e:
                messagebox.showerror("Error al descargar imagen", f"Error con la imagen: {url}\n{str(e)}")
            self.progress['value'] = (index + 1) / total * 100
            self.root.update_idletasks()
    
    def preview_image(self):
        # Verifica si hay datos en `self.image_data` y si `self.current_index` es válido
        if not self.image_data or self.current_index >= len(self.image_data):
            # No mostramos el mensaje de advertencia en caso de que las imágenes aún no se hayan descargado
            return  # Salimos sin hacer nada si no hay imágenes para mostrar

        # Procede con la lógica de previsualización si todo está bien
        image_info = self.image_data[self.current_index]

        # Obtener la ruta local de la imagen
        image_path = image_info.get('local_path')
        if image_path:
            try:
                # Cargar y mostrar la imagen en el canvas
                img = Image.open(image_path)
                img = ImageTk.PhotoImage(img)
                self.canvas.create_image(0, 0, image=img, anchor='nw')
                self.canvas.image = img  # Referencia para mantener la imagen en memoria
            except Exception as e:
                messagebox.showerror("Error", f"No se pudo cargar la imagen de previsualización: {str(e)}")

        # Crear una imagen en blanco para la vista previa
        preview_image = Image.new("RGB", (self.image_width.get(), self.image_height.get()), (255, 255, 255))
        draw = ImageDraw.Draw(preview_image)

        # Dibujar el fondo si está cargado
        if self.bg_image_path:
            try:
                bg_image = Image.open(self.bg_image_path)
                bg_image = bg_image.resize((self.image_width.get(), self.image_height.get()))
                preview_image.paste(bg_image, (0, 0))  # Asegurarse de que el fondo cubra toda la imagen
            except Exception as e:
                messagebox.showerror("Error", f"No se pudo cargar la plantilla de fondo: {e}")
                return

        # Cargar la imagen del producto si 'local_path' está disponible en `image_data`
        if 'local_path' in self.image_data[self.current_index]:
            try:
                product_image = Image.open(self.image_data[self.current_index]['local_path'])
                product_image = product_image.resize((self.image_width.get(), self.image_height.get()))  # Redimensionar si es necesario
                preview_image.paste(product_image, (0, 0))  # Poner la imagen sobre el fondo
            except Exception as e:
                messagebox.showerror("Error", f"No se pudo cargar la imagen del producto: {e}")
                return


        # Dibujar el título y precio sobre la plantilla de fondo
        title_font = ImageFont.truetype(self.font_path, self.title_size.get())
        price_font = ImageFont.truetype(self.font_path, self.price_size.get())

        title_text = self.image_data[self.current_index]['Título']
        price_text = f"${self.image_data[self.current_index]['Precio']}"

        # Colores
        title_color = self.title_color_display.cget("bg")
        price_color = self.price_color_display.cget("bg")

        # Dibujar el título y el precio
        draw.text(self.title_position, title_text, fill=title_color, font=title_font)
        draw.text(self.price_position, price_text, fill=price_color, font=price_font)

        # Mostrar la vista previa en el lienzo
        preview_image_tk = ImageTk.PhotoImage(preview_image)
        self.canvas.create_image(0, 0, image=preview_image_tk, anchor=NW)
        self.canvas.image = preview_image_tk  # Necesario para evitar que la imagen se borre


    def create_preview(self, data):
        try:
            image_path = data['local_path']
            title_text = data['Título']
            price_text = str(data['Precio'])

            # Cargar imagen del producto y ajustar tamaño
            original_image = Image.open(image_path).convert("RGBA")
            aspect_ratio = original_image.width / original_image.height
            new_width = int(self.image_width.get() * self.zoom_level)
            new_height = int(new_width / aspect_ratio)
            original_image = original_image.resize((new_width, new_height))

            # Aplicar fondo si está seleccionado
            if self.bg_image_path:
                bg_image = Image.open(self.bg_image_path).convert("RGBA")
                bg_image = bg_image.resize((new_width, new_height))
                original_image = Image.alpha_composite(bg_image, original_image)

            # Aplicar filtro seleccionado (corregido)
            if self.filters.get() != "None":
                filter_name = self.filters.get()
                original_image = original_image.filter(getattr(ImageFilter, filter_name))

            # Crear objeto de dibujo y agregar texto
            draw = ImageDraw.Draw(original_image)
            title_font = ImageFont.truetype(self.font_path, int(self.title_size.get() * self.zoom_level))
            price_font = ImageFont.truetype(self.font_path, int(self.price_size.get() * self.zoom_level))
            draw.text(self.title_position, title_text, font=title_font, fill=self.title_color_display.cget("bg"))
            draw.text(self.price_position, price_text, font=price_font, fill=self.price_color_display.cget("bg"))

            # Mostrar la vista previa
            self.preview_image_tk = ImageTk.PhotoImage(original_image)
            self.canvas.create_image(0, 0, anchor=NW, image=self.preview_image_tk)
            self.canvas.config(scrollregion=self.canvas.bbox(ALL))

        except Exception as e:
            messagebox.showerror("Error en la previsualización", str(e))



    def next_image(self):
        if self.current_index < len(self.image_data) - 1:
            self.current_index += 1
            self.preview_image()

    def previous_image(self):
        if self.current_index > 0:
            self.current_index -= 1
            self.preview_image()

    def save_images(self):
        if not self.image_data:
            messagebox.showerror("Error", "No hay imágenes para guardar.")
            return

        output_dir = filedialog.askdirectory()
        if not output_dir:
            return

        for index, data in enumerate(self.image_data):
            try:
                image_path = data['local_path']
                title_text = data['Título']
                price_text = str(data['Precio'])  # Convertir el precio a string

                original_image = Image.open(image_path).convert("RGBA")
                original_image = original_image.resize((self.image_width.get(), self.image_height.get()))

                if self.bg_image_path:
                    bg_image = Image.open(self.bg_image_path).convert("RGBA")
                    bg_image = bg_image.resize((self.image_width.get(), self.image_height.get()))
                    original_image = Image.alpha_composite(bg_image, original_image)

                # Aplicar filtro seleccionado
                filter_name = self.filters.get()
                if filter_name == "BLUR":
                    original_image = original_image.filter(ImageFilter.BLUR)
                elif filter_name == "CONTOUR":
                    original_image = original_image.filter(ImageFilter.CONTOUR)
                elif filter_name == "DETAIL":
                    original_image = original_image.filter(ImageFilter.DETAIL)
                elif filter_name == "EDGE_ENHANCE":
                    original_image = original_image.filter(ImageFilter.EDGE_ENHANCE)
                elif filter_name == "SHARPEN":
                    original_image = original_image.filter(ImageFilter.SHARPEN)

                draw = ImageDraw.Draw(original_image)
                title_font = ImageFont.truetype(self.font_path, self.title_size.get())
                price_font = ImageFont.truetype(self.font_path, self.price_size.get())

                title_position = (50, 50)
                price_position = (50, 100)

                draw.text(title_position, title_text, font=title_font, fill=self.title_color_display.cget("bg"))
                draw.text(price_position, price_text, font=price_font, fill=self.price_color_display.cget("bg"))

                save_path = os.path.join(output_dir, f"post_{index + 1}.png")
                original_image.save(save_path)
            except Exception as e:
                messagebox.showerror("Error al guardar la imagen", f"Error con la imagen: {data['URL']}\n{str(e)}")

        messagebox.showinfo("Éxito", "Imágenes guardadas correctamente.")

    def process_all_images(self):
        if not self.image_data:
            messagebox.showerror("Error", "No hay imágenes para procesar.")
            return

        for index, data in enumerate(self.image_data):
            try:
                image_path = data['local_path']
                title_text = data['Título']
                price_text = str(data['Precio'])  # Convertir el precio a string

                original_image = Image.open(image_path).convert("RGBA")
                original_image = original_image.resize((self.image_width.get(), self.image_height.get()))

                if self.bg_image_path:
                    bg_image = Image.open(self.bg_image_path).convert("RGBA")
                    bg_image = bg_image.resize((self.image_width.get(), self.image_height.get()))
                    original_image = Image.alpha_composite(bg_image, original_image)

                # Aplicar filtro seleccionado
                filter_name = self.filters.get()
                if filter_name == "BLUR":
                    original_image = original_image.filter(ImageFilter.BLUR)
                elif filter_name == "CONTOUR":
                    original_image = original_image.filter(ImageFilter.CONTOUR)
                elif filter_name == "DETAIL":
                    original_image = original_image.filter(ImageFilter.DETAIL)
                elif filter_name == "EDGE_ENHANCE":
                    original_image = original_image.filter(ImageFilter.EDGE_ENHANCE)
                elif filter_name == "SHARPEN":
                    original_image = original_image.filter(ImageFilter.SHARPEN)

                draw = ImageDraw.Draw(original_image)
                title_font = ImageFont.truetype(self.font_path, self.title_size.get())
                price_font = ImageFont.truetype(self.font_path, self.price_size.get())

                title_position = (50, 50)
                price_position = (50, 100)

                draw.text(title_position, title_text, font=title_font, fill=self.title_color_display.cget("bg"))
                draw.text(price_position, price_text, font=price_font, fill=self.price_color_display.cget("bg"))

                save_path = os.path.join(self.output_dir, f"post_{index + 1}.png")
                original_image.save(save_path)
            except Exception as e:
                messagebox.showerror("Error al procesar la imagen", f"Error con la imagen: {data['URL']}\n{str(e)}")

        messagebox.showinfo("Éxito", "Todas las imágenes han sido procesadas correctamente.")

    def choose_title_color(self):
        color = colorchooser.askcolor()[1]
        if color:
            self.title_color_display.config(bg=color)

    def choose_price_color(self):
        color = colorchooser.askcolor()[1]
        if color:
            self.price_color_display.config(bg=color)

    def apply_preset(self, event):
        preset = self.presets.get()
        if preset == "Facebook Post":
            self.image_width.set(1200)
            self.image_height.set(630)
        elif preset == "Facebook Carousel":
            self.image_width.set(1080)
            self.image_height.set(1080)
        elif preset == "Instagram Post":
            self.image_width.set(1080)
            self.image_height.set(1080)
        elif preset == "Instagram Story":
            self.image_width.set(1080)
            self.image_height.set(1920)

    def update_zoom(self, value):
        self.zoom_level = float(value)
        self.preview_image()

    def select_output_dir(self):
        directory = filedialog.askdirectory()
        if directory:
            self.output_dir = directory
            self.output_entry.delete(0, END)
            self.output_entry.insert(0, self.output_dir)

    def undo_changes(self):
        self.image_data = []
        self.current_index = 0
        self.bg_image_path = None
        self.bg_entry.delete(0, END)
        self.image_label.config(image=None)
        self.output_entry.delete(0, END)
        self.output_entry.insert(0, os.getcwd())

 # Nueva función para agregar una capa
    def add_layer(self, name, image):
        layer = Layer(name, image)
        self.layer_manager.add_layer(layer)
        self.update_layer_list()

    # Nueva función para eliminar una capa
    def remove_layer(self, index):
        self.layer_manager.remove_layer(index)
        self.update_layer_list()

# Función para guardar una imagen con opciones de formato y calidad
    def save_image(self, image, filename, format='PNG', quality=95):
        if format == 'JPEG':
            image.save(filename, format='JPEG', quality=quality)
        else:
            image.save(filename, format='PNG')

    # Función para programar la generación de publicaciones
    def schedule_task(self, interval, task):
        schedule.every(interval).minutes.do(task)
        while True:
            schedule.run_pending()
            time.sleep(1)
    
from PIL import Image, ImageTk
import tkinter as tk

# Clase para capas
class Layer:
    def __init__(self, name, image, opacity=1.0, visible=True):
        self.name = name
        self.image = image  # Image object (Pillow)
        self.opacity = opacity
        self.visible = visible
        self.x = 0  # Posición X en el canvas
        self.y = 0  # Posición Y en el canvas

    def set_opacity(self, opacity):
        self.opacity = opacity

    def set_visibility(self, visible):
        self.visible = visible

    def render(self):
        layer_image = self.image.convert("RGBA")
        layer_image.putalpha(int(self.opacity * 255))
        return layer_image

# Clase para manejar capas
class LayerManager:
    def __init__(self):
        self.layers = []

    def add_layer(self, layer):
        self.layers.append(layer)

    def remove_layer(self, layer):
        self.layers.remove(layer)

    def move_layer_up(self, layer):
        index = self.layers.index(layer)
        if index < len(self.layers) - 1:
            self.layers[index], self.layers[index + 1] = self.layers[index + 1], self.layers[index]

    def move_layer_down(self, layer):
        index = self.layers.index(layer)
        if index > 0:
            self.layers[index], self.layers[index - 1] = self.layers[index - 1], self.layers[index]

    def update_layer_opacity(self, layer, opacity):
        layer.set_opacity(opacity)

    def render_layers(self):
        final_image = Image.new("RGBA", (500, 500), (255, 255, 255, 0))
        for layer in self.layers:
            if layer.visible:
                layer_image = layer.render()
                final_image.paste(layer_image, (layer.x, layer.y), layer_image)
        return final_image

# Clase para la aplicación de control de capas
class LayerControlApp:
    def __init__(self, root, layer_manager):
        self.layer_manager = layer_manager
        self.root = root
        self.root.title("Gestión de Capas")

        # Configuración del canvas
        self.canvas = tk.Canvas(self.root, width=500, height=500)
        self.canvas.grid(row=0, column=0, columnspan=4)

        self.layer_listbox = tk.Listbox(self.root, height=10, width=50)
        self.layer_listbox.grid(row=1, column=0, columnspan=4)
        self.layer_listbox.bind("<ButtonRelease-1>", self.select_layer)

        self.opacity_label = tk.Label(self.root, text="Opacidad:")
        self.opacity_label.grid(row=2, column=0)

        self.opacity_slider = tk.Scale(self.root, from_=0, to=100, orient="horizontal", command=self.update_opacity)
        self.opacity_slider.set(100)  # Valor predeterminado al 100%
        self.opacity_slider.grid(row=2, column=1)

        self.up_button = tk.Button(self.root, text="Mover Arriba", command=self.move_layer_up)
        self.up_button.grid(row=3, column=0)

        self.down_button = tk.Button(self.root, text="Mover Abajo", command=self.move_layer_down)
        self.down_button.grid(row=3, column=1)

        self.remove_button = tk.Button(self.root, text="Eliminar Capa", command=self.remove_layer)
        self.remove_button.grid(row=3, column=2)

        self.add_layer_button = tk.Button(self.root, text="Añadir Capa", command=self.add_layer)
        self.add_layer_button.grid(row=3, column=3)

        self.selected_layer = None  # Para almacenar la capa seleccionada
        self.offset_x = 0
        self.offset_y = 0

        self.canvas.bind("<Button-1>", self.on_click)
        self.canvas.bind("<B1-Motion>", self.on_drag)

    def select_layer(self, event):
        selected_index = self.layer_listbox.curselection()
        if selected_index:
            layer = self.layer_manager.layers[selected_index[0]]
            self.opacity_slider.set(layer.opacity * 100)
            self.selected_layer = layer  # Establecer la capa seleccionada

    def move_layer_up(self):
        selected_index = self.layer_listbox.curselection()
        if selected_index:
            layer = self.layer_manager.layers[selected_index[0]]
            self.layer_manager.move_layer_up(layer)
            self.update_layer_list()
            self.render_canvas()

    def move_layer_down(self):
        selected_index = self.layer_listbox.curselection()
        if selected_index:
            layer = self.layer_manager.layers[selected_index[0]]
            self.layer_manager.move_layer_down(layer)
            self.update_layer_list()
            self.render_canvas()

    def remove_layer(self):
        selected_index = self.layer_listbox.curselection()
        if selected_index:
            layer = self.layer_manager.layers[selected_index[0]]
            self.layer_manager.remove_layer(layer)
            self.update_layer_list()
            self.render_canvas()

    def add_layer(self):
        # Cargar una imagen desde el archivo
        image_path = "path/to/image.png"
        try:
            image = Image.open(image_path).convert("RGBA")
            layer = Layer(name=f"Capa {len(self.layer_manager.layers) + 1}", image=image)
            self.layer_manager.add_layer(layer)
            self.update_layer_list()
            self.render_canvas()
        except FileNotFoundError:
            print(f"El archivo no se encontró: {image_path}")

    def update_layer_list(self):
        self.layer_listbox.delete(0, tk.END)
        for layer in self.layer_manager.layers:
            self.layer_listbox.insert(tk.END, layer.name)

    def update_opacity(self, val):
        selected_index = self.layer_listbox.curselection()
        if selected_index:
            layer = self.layer_manager.layers[selected_index[0]]
            opacity = self.opacity_slider.get() / 100.0
            self.layer_manager.update_layer_opacity(layer, opacity)
            self.render_canvas()

    def render_canvas(self):
        # Limpiar el canvas
        self.canvas.delete("all")
        for layer in self.layer_manager.layers:
            if layer.visible:
                layer_image = layer.render()
                layer_image_tk = ImageTk.PhotoImage(layer_image)
                self.canvas.create_image(layer.x, layer.y, image=layer_image_tk, anchor="nw")

    def on_click(self, event):
        # Guardar la posición inicial cuando se hace clic sobre una capa
        self.selected_layer = None
        selected_index = self.layer_listbox.curselection()
        if selected_index:
            layer = self.layer_manager.layers[selected_index[0]]
            self.selected_layer = layer
            self.offset_x = event.x - layer.x
            self.offset_y = event.y - layer.y

    def on_drag(self, event):
        # Mover la capa seleccionada con el mouse
        if self.selected_layer:
            new_x = event.x - self.offset_x
            new_y = event.y - self.offset_y
            self.selected_layer.x = new_x
            self.selected_layer.y = new_y
            self.render_canvas()  # Actualizar el canvas con la nueva posición

# Iniciar la aplicación principal
root = tk.Tk()
app = ImagePostApp(root)
root.mainloop()
