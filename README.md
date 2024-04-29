# Bizcard_Extaraction_project
%%writefile app.py

import streamlit as st
from streamlit_option_menu import option_menu
from PIL import Image
import pandas as pd
import numpy as np
import easyocr
import re
import io
import sqlite3


def image_to_text(path):

  input_img=Image.open(path)

  #converting image to array format
  image_arr= np.array(input_img)

  reader= easyocr.Reader(['en'])
  text= reader.readtext(image_arr, detail=0)

  return text,input_img


def extracted_text(texts):

  extrd_dict = {"Name":[], "DESIGNATION":[], "COMPANYNAME":[], "CONTACT":[], "EMAIL":[], "WEBSITE":[],
                "ADDRESS":[], "PINCODE":[]}

  extrd_dict["Name"].append(texts[0])
  extrd_dict["DESIGNATION"].append(texts[1])


  for i in range(2,len(texts)):
    if texts[i].startswith("+") or (texts[i].replace("-","").isdigit() and '-' in texts[i]):
      extrd_dict["CONTACT"].append(texts[i])

    elif "@" in texts[i] and ".com" in texts[i]:
      extrd_dict["EMAIL"].append(texts[i])

    elif "WWW" in texts[i] or "www" in texts[i] or "Www" in texts[i] or "wWw" in texts[i] or "wwW" in texts[i]:
      #small = texts[i].lower()
      extrd_dict["WEBSITE"].append(texts[i].lower())

    elif "Tamil Nadu" in texts[i] or "TamilNadu" in texts[i] or texts[i].isdigit():
      extrd_dict["PINCODE"].append(texts[i])

    elif re.match(r'^[A-Za-z]', texts[i]):
      extrd_dict["COMPANYNAME"].append(texts[i])

    else:
      remove_colon = re.sub(r'[,;]','',texts[i])
      extrd_dict["ADDRESS"].append(remove_colon)

  for key,value in extrd_dict.items():
    if len(value)>0:
      concadenate=" ".join(value)
      extrd_dict[key]=[concadenate]

    else:
      value= "NA"
      extrd_dict[key] = [value]

  return extrd_dict


#streamlit part
st.set_page_config(layout = "wide")
st.title("EXTRACTING BIZCARD WITH OCR")
st.image("/content/project.webp")

with st.sidebar:

  select=option_menu("Menu",["About the project","Upload and Modifying", "Delete"])


if select == "About the project":
  st.markdown("#### Project description")
  st.write("It have been tasked with developing a Streamlit application that allows users to upload an image of a business card and extract relevant information from it using easyOCR. The extracted information should include the company name, card holder name, designation, mobile number, email address, website URL, area, city, state, and pin code. The extracted information should then be displayed in the application's graphical user interface (GUI).")
  st.markdown("#### Skills Required:")
  st.write("The project would require skills in image processing, OCR, GUI development, and database management. It would also require careful design and planning of the application architecture to ensure that it is scalable, maintainable, and extensible.   Good documentation and code organization would also be important for the project.Overall, the result of the project would be a useful tool for businesses and individuals who need to manage business card information efficiently.")
  st.markdown("#### :green[Technologies used :] Github, OCR, Python, Pandas, SQL, GUI, Streamlit.")
  col1,col2,col3,col4,col5=st.columns(5,gap="small")
  with col1:

      st.image("/content/github.png")
  with col2:

      st.image("/content/python.png")

  with col3:

      st.image("/content/sqllite.jpg")

  with col4:

      st.image("/content/streamlit.png")

  with col5:

      st.image("/content/imgtotxt.png")


elif select == "Upload and Modifying":
  img = st.file_uploader("Upload the Image", type=["png","jpg","jpeg"])

  if img is not None:
    st.image(img, width=400)

    text_image, input_img = image_to_text(img)

    text_dict = extracted_text(text_image)

    if text_dict:
      st.success("Text is extracted successfully")

    df= pd.DataFrame(text_dict)


    #converting Image to Bytes

    Image_bytes = io.BytesIO()
    input_img.save(Image_bytes, format= "PNG")

    image_data = Image_bytes.getvalue()


    #creating Dictionary
    data = {"IMAGE":[image_data]}

    df_1 = pd.DataFrame(data)

    concat_df= pd.concat([df,df_1],axis= 1)
    st.dataframe(concat_df)

    button_1 = st.button("Save", use_container_width= True)

    if button_1:


      mydb = sqlite3.connect("Bizcardx.db")
      cursor = mydb.cursor()

      #Table Creations
      create_table_query = '''CREATE TABLE IF NOT EXISTS bizcard_details(name varchar(225),
                                                                          designation varchar(225),
                                                                          company_name varchar(225),
                                                                          contact varchar(225),
                                                                          email varchar(225),
                                                                          website text,
                                                                          address text,
                                                                          pincode varchar(225),
                                                                          image text)'''

      cursor.execute(create_table_query)
      mydb.commit()

      #Inset Query

      insert_query = '''INSERT INTO bizcard_details(name, designation, company_name, contact, email, website, address, pincode, image)

                                                      values(?,?,?,?,?,?,?,?,?)'''


      datas = concat_df.values.tolist()[0]
      cursor.execute(insert_query,datas)
      mydb.commit()
      
      st.success("Saved successfully")
  method = st.radio("select the method",["none","preview","modify"])

  if method == "none":
    st.write("")

  if method == "preview":

    mydb = sqlite3.connect("Bizcardx.db")
    cursor = mydb.cursor()

    #select query
    select_query="SELECT * FROM bizcard_details"
    cursor.execute(select_query)
    table = cursor.fetchall()
    mydb.commit()

    table_df = pd.DataFrame(table, columns=("Name", "DESIGNATION", "COMPANY_NAME", "CONTACT", "EMAIL", "WEBSITE","ADDRESS", "PINCODE", "IMAGE"))
    st.dataframe(table_df)

  elif method == "modify":

    mydb = sqlite3.connect("Bizcardx.db")
    cursor = mydb.cursor()

    #select query
    select_query="SELECT * FROM bizcard_details"
    cursor.execute(select_query)
    table = cursor.fetchall()
    mydb.commit()

    table_df = pd.DataFrame(table, columns=("Name", "DESIGNATION", "COMPANY_NAME", "CONTACT", "EMAIL", "WEBSITE","ADDRESS", "PINCODE", "IMAGE"))
    col1,col2 = st.columns(2)
    with col1:
      selected_name=st.selectbox("Select the name",table_df["Name"])

    df_3 = table_df[table_df["Name"]== selected_name]

    df_4 = df_3.copy()

    col1,col2 = st.columns(2)
    with col1:
      no_name = st.text_input("Name",df_3["Name"].unique()[0])
      no_desi = st.text_input("DESIGNATION",df_3["DESIGNATION"].unique()[0])
      no_com_name = st.text_input("COMPANY_NAME",df_3["COMPANY_NAME"].unique()[0])
      no_contact = st.text_input("CONTACT",df_3["CONTACT"].unique()[0])
      no_email = st.text_input("EMAIL",df_3["EMAIL"].unique()[0])

      df_4["Name"] = no_name
      df_4["DESIGNATION"] = no_desi
      df_4["COMPANY_NAME"] = no_com_name
      df_4["CONTACT"] = no_contact
      df_4["EMAIL"] = no_email

    with col2:
      no_website = st.text_input("WEBSITE",df_3["WEBSITE"].unique()[0])
      no_addre = st.text_input("ADDRESS",df_3["ADDRESS"].unique()[0])
      no_pincode = st.text_input("PINCODE",df_3["PINCODE"].unique()[0])
      no_image = st.text_input("IMAGE",df_3["IMAGE"].unique()[0])


      df_4["WEBSITE"] = no_website
      df_4["ADDRESS"] = no_addre
      df_4["PINCODE"] = no_pincode
      df_4["IMAGE"] = no_image

    st.dataframe(df_4)

    col1,col2 = st.columns(2)
    with col1:
      button_3 = st.button("modify", use_container_width = True)

    if button_3:
      mydb = sqlite3.connect("Bizcardx.db")
      cursor = mydb.cursor()

      cursor.execute(f"DELETE FROM bizcard_details WHERE Name = '{selected_name}'")
      mydb.commit()

     #Inset Query

      insert_query = '''INSERT INTO bizcard_details(name, designation, company_name, contact, email, website, address, pincode, image)

                                                      values(?,?,?,?,?,?,?,?,?)'''


      datas = df_4.values.tolist()[0]
      cursor.execute(insert_query,datas)
      mydb.commit()

      st.success("Modified successfully")

elif select == "Delete":
  mydb = sqlite3.connect("Bizcardx.db")
  cursor = mydb.cursor()
 
 
  col1,col2 = st.columns(2)
  with col1:
    select_query="SELECT Name FROM bizcard_details"
    cursor.execute(select_query)
    table1 = cursor.fetchall()
    mydb.commit()


    names = []
    for i in table1:
      names.append(i[0])

    name_select = st.selectbox("Select the name",names)

  with col2:
    select_query="SELECT DESIGNATION FROM bizcard_details"
    cursor.execute(select_query)
    table1 = cursor.fetchall()
    mydb.commit()


    designation = []
    for j in table1:
      designation.append(j[0])

    designation_select = st.selectbox("Select the designation",designation)


  remove = st.button("Delete", use_container_width=True)

  if remove:

    cursor.execute(f"DELETE FROM bizcard_details WHERE Name = '{name_select}' AND DESIGNATION = '{designation_select}'")
    mydb.commit()   

    st.success("Deleted succesfully")  
