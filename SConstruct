#!/usr/bin/env python
import os
import sys
import json
from pathlib import Path
import clang.cindex
from concurrent.futures import ThreadPoolExecutor, as_completed
import threading
import time

clang.cindex.Config.set_library_file(r'C:/Users/ernes/scoop/apps/llvm/current/bin/libclang.dll')

def find_files_recursively(dir, glob_pattern):
	return [str(path.relative_to(dir)) for path in Path(dir).rglob(glob_pattern)]

def parse_class(cursor, processed_classes):
	class_name = cursor.spelling
	if class_name in processed_classes:
		return None
	
	processed_classes.add(class_name)
	
	class_info = {
		"name": cursor.spelling,
		"methods": [],
		"base_classes": [],
		"enums": [],
		"structs": [],
		"classes": []
	}

	print(f"Processing class: {cursor.spelling}")

	for c in cursor.get_children():
		if c.kind == clang.cindex.CursorKind.CXX_BASE_SPECIFIER:
			base_class_name = c.type.spelling
			class_info["base_classes"].append(base_class_name)
		elif c.kind == clang.cindex.CursorKind.CXX_METHOD:
			if c.spelling == "GDEXTENSION_CLASS":
				continue
			method_info = {
				"name": c.spelling,
				"return_type": c.result_type.spelling,
				"arguments": [{"name": arg.spelling, "type": arg.type.spelling} for arg in c.get_arguments()],
				"virtual": c.is_virtual_method(),
				"static": c.is_static_method(),
				"constructor": c.kind == clang.cindex.CursorKind.CONSTRUCTOR,
				"destructor": c.kind == clang.cindex.CursorKind.DESTRUCTOR
			}
			class_info["methods"].append(method_info)
		elif c.kind == clang.cindex.CursorKind.ENUM_DECL:
			class_info["enums"].append(parse_enum(c))
		elif c.kind == clang.cindex.CursorKind.STRUCT_DECL:
			struct_info = parse_class(c, processed_classes)
			if struct_info:
				class_info["structs"].append(struct_info)
		elif c.kind == clang.cindex.CursorKind.CLASS_DECL and not c.is_definition():
			nested_class_info = parse_class(c, processed_classes)
			if nested_class_info:
				class_info["classes"].append(nested_class_info)

	return class_info

def parse_enum(cursor):
	enum_info = {
		"name": cursor.spelling,
		"values": []
	}
	for c in cursor.get_children():
		if c.kind == clang.cindex.CursorKind.ENUM_CONSTANT_DECL:
			enum_info["values"].append({"name": c.spelling, "value": c.enum_value})
	return enum_info

def parse_header_file(file_path, include_paths, processed_classes):
	index = clang.cindex.Index.create()
	args = ['-I' + path for path in include_paths]
	try:
		translation_unit = index.parse(file_path, args=args)
	except clang.cindex.TranslationUnitLoadError as e:
		print(f"Error parsing file {file_path}: {e}")
		return None

	classes = []
	structs = []
	enums = []

	for cursor in translation_unit.cursor.get_children():
		if cursor.kind == clang.cindex.CursorKind.NAMESPACE and cursor.spelling == "godot":
			for child in cursor.get_children():
				if child.kind == clang.cindex.CursorKind.CLASS_DECL and child.is_definition():
					class_info = parse_class(child, processed_classes)
					if class_info:
						classes.append(class_info)
				elif child.kind == clang.cindex.CursorKind.STRUCT_DECL and child.is_definition():
					struct_info = parse_class(child, processed_classes)
					if struct_info:
						structs.append(struct_info)
				elif child.kind == clang.cindex.CursorKind.ENUM_DECL:
					enums.append(parse_enum(child))

	return {"classes": classes, "structs": structs, "enums": enums}

input_folders = [
	"thirdparty/godot-sandbox/godot-cpp/gen/include",
	"thirdparty/godot-sandbox/godot-cpp/include"
]

include_paths = [
	"thirdparty/godot-sandbox/godot-cpp/gen/include",
	"thirdparty/godot-sandbox/godot-cpp/include",
]

input_files = []
for folder in input_folders:
	input_files.extend(find_files_recursively(folder, "*.hpp"))

processed_classes = set()

def process_file(input_file):
	return input_file, parse_header_file(os.path.join(input_folders[0], input_file), include_paths, processed_classes)

file_classes_dict = {}

def write_json_periodically(file_classes_dict, interval=60):
	while True:
		with open("godot_sandbox_api.json", "w") as file:
			json.dump(file_classes_dict, file, indent=4)
		time.sleep(interval)

json_writer_thread = threading.Thread(target=write_json_periodically, args=(file_classes_dict,))
json_writer_thread.daemon = True
json_writer_thread.start()

with ThreadPoolExecutor() as executor:
	future_to_file = {executor.submit(process_file, input_file): input_file for input_file in input_files if not input_file.startswith(("thirdparty/", "tests/"))}
	for future in as_completed(future_to_file):
		input_file, result = future.result()
		if result is not None:
			file_classes_dict[input_file] = result

with open("godot_sandbox_api.json", "w") as file:
	json.dump(file_classes_dict, file, indent=4)

env = Environment()

SConscript("thirdparty/godot-sandbox/godot-cpp/SConstruct", exports='env')
