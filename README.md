using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Data;
using System.Drawing;
using System.Linq;
using System.Text;
using System.IO;
using System.Threading.Tasks;
using System.Windows.Forms;
using Newtonsoft.Json;
using Translate_Java;
using static Translate_Java.JsonData;

namespace Translate_Java
{
    public partial class Form1 : Form
    {

        private readonly StringBuilder sourceCodeBuilder;
        private string selectedJsonFilePath;
        private string ConstructName;

        public Form1()
        {
            InitializeComponent();
            sourceCodeBuilder = new StringBuilder();
        }

        private void GenerateJava(string jsonFilePath)
        {
            string umlDiagramJson = File.ReadAllText(jsonFilePath);

            // Decode JSON data
            JsonData json = JsonConvert.DeserializeObject<JsonData>(umlDiagramJson);

            // Generate into Java 
            GeneratePackage(json.sub_name);

            foreach (var model in json.model)
            {
                if (model.type == "class")
                {
                    GenerateClass(model);
                }
                else if (model.type == "association" && model.model != null)
                {
                    GenerateAssociationClass(model.model);
                }
                if (model.type == "imported_class")
                {
                    sourceCodeBuilder.AppendLine($"//Imported Class");
                    GenerateImportedClass(model);
                }
            }

            bool generateAssocClass = json.model.Any(model => model.type == "association");
            //bool generateAssoc = json.model.Any(model => model.type == "association");

            //if (generateAssocClass)
            //{
            //    sourceCodeBuilder.AppendLine($"// Generate Association Class");
            //    GenerateAssocClass();
            //}


            //foreach (var model in json.model)
            //{
            //    if (model.type == "association")
            //    {
            //        GenerateObjAssociation(model);
            //    }
            //}

            //if (generateAssoc)
            //{
            //    GenerateAssoc();
            //}

            // Display or save the generated Java code
            richTextBox2.Text = sourceCodeBuilder.ToString();
        }
        private void GeneratePackage(string packageName)
        {
            sourceCodeBuilder.AppendLine($"package {packageName};\n");
        }

        private void GenerateClass(JsonData.Model model)
        {
            sourceCodeBuilder.AppendLine($"public class {model.class_name} {{");
            this.ConstructName = model.class_name;

            var sortedAttributes = model.attributes.OrderBy(attr => attr.attribute_name);

            foreach (var attribute in model.attributes)
            {
                GenerateAttribute(attribute);
            }

            sourceCodeBuilder.AppendLine("");

            //foreach (var status in model.attributes)
            //{
            //    GenerateState(status);
            //}

            //sourceCodeBuilder.AppendLine("");

            if (model.attributes != null)
            {
                GenerateConstructor(model.attributes);
            }

            sourceCodeBuilder.AppendLine("");

            foreach (var attribute in model.attributes)
            {
                GenerateGetter(attribute);
            }

            sourceCodeBuilder.AppendLine("");

            foreach (var attribute in model.attributes)
            {
                GenerateSetter(attribute);
            }

            sourceCodeBuilder.AppendLine("");

            if (model.states != null)
            {
                
                    GenerateStateTransitionMethod(model.states);

                foreach (var stateAttribute in model.attributes.Where(attr => attr.data_type == "state"))
                {
                    GenerateGetState(stateAttribute);
                }
                
            }

            //if (model.states != null)
            //{
            //    GenerateGetState();
            //}


            sourceCodeBuilder.AppendLine("}\n\n");
        }
        private void GenerateAttribute(JsonData.Attribute1 attribute)
        {
            // Adjust data types as needed
            string dataType = MapDataType(attribute.data_type);

            //if (dataType != "state")
            if (attribute.data_type != "state")
            {
                sourceCodeBuilder.AppendLine($"    private {dataType} {attribute.attribute_name};");
            }
            else
            {
                sourceCodeBuilder.AppendLine($"    private {attribute.attribute_name};");
            }

        }

        //private void GenerateState(JsonData.Attribute1 status)
        //{
        //    if (status.attribute_name == "status")
        //    {
        //        sourceCodeBuilder.AppendLine("    private String state;");
        //    }
        //}

        private void GenerateAssociationClass(JsonData.Model associationModel)
        {
            // Check if associationModel is not null
            if (associationModel == null)
            {
                // Handle the case where associationModel is null, e.g., throw an exception or log a message
                return;
            }

            sourceCodeBuilder.AppendLine($"public class assoc_{associationModel.class_name} {{");

            foreach (var attribute in associationModel.attributes)
            {
                // Adjust data types as needed
                string dataType = MapDataType(attribute.data_type);

                sourceCodeBuilder.AppendLine($"     private {dataType} {attribute.attribute_name};");
            }

            // Check if associatedClass.@class is not null before iterating
            if (associationModel.@class != null)
            {
                foreach (var associatedClass in associationModel.@class)
                {
                    if (associatedClass.class_multiplicity == "1..1")
                    {
                        sourceCodeBuilder.AppendLine($"    private {associatedClass.class_name} {associatedClass.class_name};");
                    }
                    else
                    {
                        sourceCodeBuilder.AppendLine($"    private array {associatedClass.class_name}List;");
                    }
                }
            }

            sourceCodeBuilder.AppendLine("");

            if (associationModel.attributes != null)
            {
                GenerateConstructor(associationModel.attributes);
            }

            foreach (var attribute in associationModel.attributes)
            {
                GenerateGetter(attribute);
            }

            foreach (var attribute in associationModel.attributes)
            {
                GenerateSetter(attribute);
            }
            sourceCodeBuilder.AppendLine("}\n\n");
        }

        private void GenerateImportedClass(JsonData.Model imported)
        {
            if (imported == null)
            {
                return;
            }
            sourceCodeBuilder.AppendLine($"class {imported.class_name} {{");

            foreach (var attribute in imported.attributes)
            {
                GenerateAttribute(attribute);
            }

            sourceCodeBuilder.AppendLine("");

            if (imported.attributes != null)
            {
                GenerateConstructor(imported.attributes);
            }

            sourceCodeBuilder.AppendLine("");

            foreach (var attribute in imported.attributes)
            {
                GenerateGetter(attribute);
            }

            sourceCodeBuilder.AppendLine("");

            foreach (var attribute in imported.attributes)
            {
                GenerateSetter(attribute);
            }

            sourceCodeBuilder.AppendLine("");

            if (imported.states != null)
            {

                GenerateStateTransitionMethod(imported.states);

                foreach (var stateAttribute in imported.attributes.Where(attr => attr.data_type == "state"))
                {
                    GenerateGetState(stateAttribute);
                }
            }
        }


        private void GenerateConstructor(List<JsonData.Attribute1> attributes)
        {
            
            sourceCodeBuilder.Append($"\tpublic {this.ConstructName} (");

            foreach (var attribute in attributes)
            {
                //if (attribute.attribute_name != "status")
                if (attribute.data_type != "state")
                {

                    string dataType = MapDataType(attribute.data_type);
                    sourceCodeBuilder.Append($"{dataType} {attribute.attribute_name}, ");
                }

            }

            // Remove the trailing comma and add the closing parenthesis
            if (attributes.Any())
            {
                sourceCodeBuilder.Length -= 1; // Remove the last character (",")
            }

            sourceCodeBuilder.AppendLine(") {");

            foreach (var attribute in attributes)
            {
                //if (attribute.attribute_name != "status")
                if (attribute.data_type != "state")
                {
                    sourceCodeBuilder.AppendLine($"        this.{attribute.attribute_name} = {attribute.attribute_name};");
                }
            }
            // Handle the "state" datatype separately outside the loop
            var stateAttribute = attributes.FirstOrDefault(attr => attr.data_type == "state");
            if (stateAttribute != null)
            {
                // Check if the attribute has a default value and it is a string
                if (!string.IsNullOrEmpty(stateAttribute.default_value) && stateAttribute.data_type.ToLower() == "state")
                {
                    int lastDotIndex = stateAttribute.default_value.LastIndexOf('.');
                    // Replace "status" with "state" and "aktif" with "active"
                    string stringValue = stateAttribute.default_value.Substring(lastDotIndex + 1);
                    sourceCodeBuilder.AppendLine($"        this.{stateAttribute.attribute_name} = \"{stringValue}\";");
                }
            }

            sourceCodeBuilder.AppendLine("}");
        }

        private void GenerateGetter(JsonData.Attribute1 getter)
        {
            //if (getter.attribute_name != "status")
            if (getter.data_type != "state")
            {
                string dataType = MapDataType(getter.data_type);
                sourceCodeBuilder.AppendLine($"      public {dataType} get{getter.attribute_name}() {{"); // ini belom diubah
                sourceCodeBuilder.AppendLine($"        this.{getter.attribute_name};");
                sourceCodeBuilder.AppendLine($"}}");
            }

        }

        private void GenerateSetter(JsonData.Attribute1 setter)
        {
            //if (setter.attribute_name != "status")
            if (setter.data_type != "state")
            {
                string dataType = MapDataType(setter.data_type);
                sourceCodeBuilder.AppendLine($"      public void set{setter.attribute_name}({dataType} {setter.attribute_name}) {{"); // ini ( String get() ) nya belom jadi
                sourceCodeBuilder.AppendLine($"        this.{setter.attribute_name} = {setter.attribute_name};");
                sourceCodeBuilder.AppendLine($"}}");
            }

        }

        private void GenerateGetState(JsonData.Attribute1 getstate)
        {
            if (getstate.data_type == "state")
            {
                sourceCodeBuilder.AppendLine($"     public string GetState() {{");  // bagian ini nya belom
                sourceCodeBuilder.AppendLine($"       this.state;");
                sourceCodeBuilder.AppendLine($"}}\n");
            }
        }

        private void GenerateStateTransitionMethod(List<JsonData.State> states)  // ini juga belomm
        {
            //if (state.state_event != null && state.state_event.Length > 0)
            foreach (var state in states)
            {
                if (state.state_event != null)
                {
                    foreach (var eventName in state.state_event)
                    {
                        string methodName = $"{Char.ToUpper(eventName[0])}{eventName.Substring(1)}";

                        sourceCodeBuilder.AppendLine($"     public void {methodName}() {{");

                        if (state.transitions != null)
                        {
                            foreach (var transition in state.transitions)
                            {
                                if (transition != null)
                                {
                                    string targetStateId = transition.target_state_id;
                                    string targetState = transition.target_state;

                                    if (!string.IsNullOrEmpty(targetStateId))
                                    {
                                        sourceCodeBuilder.AppendLine($"       this.SetStateById({targetStateId});");
                                    }
                                }
                            }
                        }

                        sourceCodeBuilder.AppendLine($"       this.{state.state_name} = \"{state.state_value}\";");
                        sourceCodeBuilder.AppendLine($"     }}\n");
                    }
                }

                //string setEvent = state.state_event[0];
                //string onEvent = state.state_event[1];
                //sourceCodeBuilder.AppendLine($"     public void {setEvent}() {{");
                //sourceCodeBuilder.AppendLine($"       this.state = \"{state.state_value}\";");
                //sourceCodeBuilder.AppendLine($"}}\n");

                //sourceCodeBuilder.AppendLine($"     public void {onEvent}() {{");
                //sourceCodeBuilder.AppendLine($"       System.out.println \"status saat ini {state.state_value}\";");
                //sourceCodeBuilder.AppendLine($"}}");
            }


            //if (state.state_function != null && state.state_function.Length > 0)  // ini juga belomm
            //{
            //    string setFunction = state.state_function[0];
            //    sourceCodeBuilder.AppendLine($"     public void {setFunction}() {{");
            //    sourceCodeBuilder.AppendLine($"       this.state = \"{state.state_value}\";");
            //    sourceCodeBuilder.AppendLine($"}}\n");
            //}
        }

        //private void GenerateAssocClass()
        //{
        //    sourceCodeBuilder.AppendLine($" public class Association{{");
        //    sourceCodeBuilder.AppendLine($"     public void ((data_type) class1, (data_type) class2) {{");
        //    sourceCodeBuilder.AppendLine($"}}");
        //    sourceCodeBuilder.AppendLine($"}}");
        //    sourceCodeBuilder.AppendLine($"\n");
        //    sourceCodeBuilder.AppendLine($" public class Main{{");
        //    sourceCodeBuilder.AppendLine($"     public static void main(String[] args){{");
        //}
        //private void GenerateObjAssociation(JsonData.Model assoc)
        //{

        //    sourceCodeBuilder.Append($"         Association {assoc.name} = new Association(");

        //    foreach (var association in assoc.@class)
        //    {
        //        sourceCodeBuilder.Append($"\"{association.class_name}\",");
        //    }

        //    sourceCodeBuilder.Length -= 1; // Remove the last character (",")

        //    sourceCodeBuilder.AppendLine($");");
        //}

        //private void GenerateAssoc()
        //{
        //    sourceCodeBuilder.AppendLine($"     }}");
        //    sourceCodeBuilder.AppendLine($"}}");
        //}

        private string MapDataType(string dataType)
        {
            switch (dataType.ToLower())
            {
                case "integer":
                    return "int";
                case "id":
                    return "int";
                case "string":
                    return "String";
                case "bool":
                    return "boolean";
                case "real":
                    return "float";
                // Add more mappings as needed
                default:
                    return dataType; // For unknown types, just pass through
            }
        }



        private void btnbrowse_Click_1(object sender, EventArgs e)
        {
            OpenFileDialog dialog = new OpenFileDialog();
            dialog.Title = "Open Json Diagram File";
            dialog.Filter = "Json Diagram Files|*.json";

            if (dialog.ShowDialog() == DialogResult.OK)
            {
                selectedJsonFilePath = dialog.FileName;
                string displayJson = File.ReadAllText(selectedJsonFilePath);
                richTextBox1.Text = displayJson;
            }

            sourceCodeBuilder.Clear();
        }

        private void btngenerate_Click_1(object sender, EventArgs e)
        {
            try
            {
                if (!string.IsNullOrEmpty(selectedJsonFilePath) && File.Exists(selectedJsonFilePath))
                {
                    sourceCodeBuilder.Clear();
                    GenerateJava(selectedJsonFilePath);
                }
                else
                {
                    MessageBox.Show("Please select a valid JSON file first.", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Error generating PHP code: {ex.Message}", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void btnexport_Click_1(object sender, EventArgs e)
        {
            SaveFileDialog saveFileDialog = new SaveFileDialog();
            saveFileDialog.Filter = "Java Files|*.java";
            saveFileDialog.Title = "Save Java File";

            if (saveFileDialog.ShowDialog() == DialogResult.OK)
            {
                string filePath = saveFileDialog.FileName;

                try
                {
                    using (StreamWriter sw = new StreamWriter(filePath))
                    {
                        sw.Write(richTextBox2.Text);
                    }

                    MessageBox.Show("File saved successfully!", "Info", MessageBoxButtons.OK, MessageBoxIcon.Information);
                }
                catch (Exception ex)
                {
                    MessageBox.Show($"Error: {ex.Message}", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
                }
            }
        }

        private void btnclear_Click(object sender, EventArgs e)
        {
            richTextBox1.Clear();
            richTextBox2.Clear();
        }

        private void btnhelp_Click(object sender, EventArgs e)
        {
            ShowHelp();
        }

        private void ShowHelp()
        {
            string helpMessage = OpenHelp();
            MessageBox.Show(helpMessage, "User Guide", MessageBoxButtons.OK, MessageBoxIcon.Information );
        }

        private string OpenHelp()
        {
            StringBuilder helpMessage = new StringBuilder();

            helpMessage.AppendLine("User Guide for Generating JSON Model into Java Programming Language");
            helpMessage.AppendLine();
            helpMessage.AppendLine("1. Open the desired JSON File by clicking Browse button");
            helpMessage.AppendLine("2. After the code appeared in the box, click the Generate button to translate it to Java");
            helpMessage.AppendLine("3. The Java Code will exist in the right box and click Export button to save it as Java file");

            return helpMessage.ToString();
        }

        private void btnparsing_Click(object sender, EventArgs e)
        {
            Parsing parsing = new Parsing();

            parsing.Show();

        }
    }
}

jadi, kode diatas ingin aku ubah agar pas klik button generate itu menerapkan fungsi parsing terlebih dahulu, kaya di cek
apakah dari json data yang diinput terdapat error atau tidak, lalu apabila aman langsung generate kode tersebut ke 
Java (sesuai kode diatas), tolong dong. sama kodenya sekalian
