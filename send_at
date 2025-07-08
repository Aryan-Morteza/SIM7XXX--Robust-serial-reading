def send_at(data):
	# Declare that we are using the global variable
	global serial_error_count; global serial_port
	if serial_error_count > 10: print('We need to restart...'); sleep(15); os.system('reboot')
	
	# List of meaningful signs (including "OK" and ">")
	meaningful_signs = ['OK', '>', 'DOWNLOAD']
	
	# Regular expression for matching any "CME/CMS ERROR: X" where X is between 0 and 100
	error_pattern = r"(CME ERROR: (?:[0-9]{1,3}|[6-7][0-9]{2}))|(CMS ERROR: (?:[0-9]{1,3}|[5-7][0-9]{2}))|ERROR"
	
	# Maximum time to wait for response
	max_wait = 45
	if data == 'AT' or data == '\x1A': max_wait = 5
	if data == 'AT+COPS=?': max_wait = 180 # Based on our experiments 
	if data.startswith("AT+COPS=") and data != 'AT+COPS=?': max_wait = 120
	if data == 'AT+COPS=2': max_wait = 45
	if data.startswith("AT+SMPUB"): max_wait = 5
	if data == 'AT+SMCONN': max_wait = 60
	timeout_flag = False
	try:
		sleep(0.1)
		response = ""
		ser = serial.Serial(serial_port, baud_rate, timeout= max_wait+5, rtscts=True, xonxoff=True,dsrdtr=True)
		sleep(0.1)
		ser.flushInput()
		ser.flushOutput()
		ser.reset_input_buffer()  # Flush old data
		ser.write((data + '\r\n').encode())
		start_time = time()
		
		# Reading serial port 
		while 1:
			if time() - start_time >= max_wait:
				timeout_flag = True
				break
			line = ser.readline().decode('utf-8', errors='replace').strip()
			response += line + "\n"  		# Append to the response
			#print(f"Received: {line}")		# Debugging: Print each line received
			if any(sign in response for sign in meaningful_signs) or re.search(error_pattern, response):
				#if '>' == response: print('Catch YA!')
				break
			sleep(0.05)
		
		# Add exception for '\x1A', replace with Ctrl+Z
		if data == '\x1A': data = 'Execute <Ctrl+Z>'
		
		# Clean up the response data
		cleaned_response = response.replace('\r\r', '\n').replace('\r\n', '\n').replace(',,', ',')  
		cleaned_response = re.sub(rf'^{re.escape(data)},?\s*', '', response, count=1).strip()

		# Check if the response contains meaningful data (not just "OK")
		if "OK" in cleaned_response.strip():
			# Only add ' ---> OK' if there's meaningful content before 'OK'
			if cleaned_response.strip() != "OK":
				cleaned_response = cleaned_response.replace('OK', '').strip() + ' ---> OK'

		# Generalize and ensure that the responses are properly separated by commas
		cleaned_response = ', '.join([line.strip() for line in cleaned_response.split('\n') if line.strip()])
		#print(cleaned_response)
		
		# Check if cleaned_response contains any of the meaningful signs or matches the ERROR pattern
		if not any(sign in cleaned_response for sign in meaningful_signs) and not re.search(error_pattern, cleaned_response):
			#print("No/Partialy response from Modem")
			if timeout_flag and not cleaned_response:
				print(f"[{data},\033[91mTimeout\033[0m]"); cleaned_response = 'Timeout'
			else:
				#print(cleaned_response)
				print(f"[{data}, \033[93m{cleaned_response}\033[0m]")				# Yellow color for partial response
		else:
			# If CME ERROR pattern is found, print it in yellow
			if re.search(error_pattern, cleaned_response):
				print(f"[{data}, \033[93m{cleaned_response}\033[0m]")				# Yellow color for errors
			else:
				if data == 'AT+SMCONN':
					print(f"[{data}, \033[92mMQTT is connected! ({cleaned_response})\033[0m]")			# Green color for clean data on MQTT
				elif data == 'AT+CSQ': 
					_, res = signal_label(cleaned_response)
					print(f"[{data}, \033[92m{cleaned_response}\033[0m ({res})]")	# Green color for clean data on signal quality
				else:
					print(f"[{data}, \033[92m{cleaned_response}\033[0m]")			# Green color for clean data
		ser.close()
		return cleaned_response
		
	except serial.serialutil.SerialException as e:
		error_msg = str(e).lower()  # Normalize error message case
		if "[errno 2]" in error_msg:
			print(f"Error: Serial port {serial_port} not found, try another port...")
			serial_error_count += 1;
		else:
			print(f"Serial error: {e}")
			serial_error_count += 1
		
	except Exception as e:
		print(f"Unexpected error: {e}")
		serial_error_count += 1
	
	finally:
		if 'ser' in locals() and ser.is_open:
			ser.close()	

	return None
