# knath28-Zinc-concentration-application
classdef ZincConcentrationApp < matlab.apps.AppBase

    % Properties that correspond to app components
    properties (Access = public)
        UIFigure                       matlab.ui.Figure
        UploadBaseImagesButton         matlab.ui.control.Button
        UploadReactionImagesButton     matlab.ui.control.Button
        BaseImagesTable                matlab.ui.control.Table
        ReactionImagesTable            matlab.ui.control.Table
        CalculateZincConcentrationButton matlab.ui.control.Button
        ZincConcentrationTextArea      matlab.ui.control.TextArea
        IntensityTextAreaBase        matlab.ui.control.TextArea
        ZincConcentrationLabel         matlab.ui.control.Label  % Add this line
        IntensityLabel         matlab.ui.control.Label  % Add this line
        StatusLabel                    matlab.ui.control.Label
        StatusTextBox                  matlab.ui.control.TextArea
        StatusLamp                     matlab.ui.control.Lamp
         ZincGauge                      matlab.ui.control.LinearGauge
         BackgroundPanel                matlab.ui.container.Panel
    end
    
    properties (Access = private)
        BaseImages     cell
        ReactionImages cell
    end

    methods (Access = private)
        
        function uploadBaseImages(app, event)
            [files, path] = uigetfile({'*.jpg;*.png;*.tif', 'Image Files'}, 'Select Base Images', 'MultiSelect', 'on');
            if iscell(files)
                app.BaseImages = cellfun(@(file) imread(fullfile(path, file)), files, 'UniformOutput', false);
                app.BaseImagesTable.Data = files';
            else
                app.BaseImages = {imread(fullfile(path, files))};
                app.BaseImagesTable.Data = {files}';
            end
        end

        function uploadReactionImages(app, event)
            [files, path] = uigetfile({'*.jpg;*.png;*.tif', 'Image Files'}, 'Select Reaction Images', 'MultiSelect', 'on');
            if iscell(files)
                app.ReactionImages = cellfun(@(file) imread(fullfile(path, file)), files, 'UniformOutput', false);
                app.ReactionImagesTable.Data = files';
            else
                app.ReactionImages = {imread(fullfile(path, files))};
                app.ReactionImagesTable.Data = {files}';
            end
        end

        function calculateZincConcentration(app, event)
            baseImages = app.BaseImages;
            reactionImages = app.ReactionImages;

            RGB_Base = [];
            for i = 1:length(baseImages)
                fprintf('welcome to base zone for image : %d  \n', i);
                [RGB_Base(i, :), ~] = app.RGBcalculator(baseImages{i});
            end
            mean_RGB_Base = mean(RGB_Base, 1);

            RGB_reaction = [];
            DE_E = [];
            for i = 1:length(reactionImages)
                fprintf('welcome to reaction zone for image : %d  \n', i);
                [RGB_reaction(i, :), ~] = app.RGBcalculator(reactionImages{i});
                DE_E(i) = (sum((mean_RGB_Base - RGB_reaction(i, :)).^2) / sum(mean_RGB_Base.^2))^0.5;
            end

            AV_DE = mean(DE_E);
            conc = (AV_DE - 0.0068) / 0.0076;
            app.ZincConcentrationTextArea.Value = sprintf('%.4f', conc);
            app.IntensityTextAreaBase.Value = sprintf('%.4f', AV_DE);
            app.ZincGauge.Value = conc;
       if conc <= 9.1
                app.StatusTextBox.Value = 'Diseased';
                app.StatusLamp.Color = 'red';
            else
                app.StatusTextBox.Value = 'Healthy';
                app.StatusLamp.Color = 'green';
            end
        
        end

        function [RGB, image] = RGBcalculator(app, image)
            figure;
            imshow(image);
            % Convert to grayscale to detect circles
            grayImage = rgb2gray(image);
            figure;
            imshow(grayImage);
            % Detect circles using the imfindcircles function
            [centers, radii, metric] = imfindcircles(grayImage, [100 130], 'ObjectPolarity', 'dark', 'Sensitivity', 0.97, 'EdgeThreshold', 0.1);
            radii = radii * 0.60;
            for i = 1:length(centers)
                viscircles(centers(i, :), radii(i), 'EdgeColor', 'b');
                text(centers(i, 1), centers(i, 2), num2str(i), 'HorizontalAlignment', 'center', 'VerticalAlignment', 'middle');
            end
            % Select the region of interest
            ROI = input("What is your region of interest:   ");
            centers = centers(ROI, :);
            radii = radii(ROI);
            % Initialize a matrix to store average RGB values
            numCircles = length(radii);
            averageRGB = zeros(numCircles, 3);
            % Loop through each detected circle
            for k = 1:numCircles
                centerX = centers(k, 1);
                centerY = centers(k, 2);
                radius = radii(k);
                % Create a mask for the current circular region
                [columnsInImage, rowsInImage] = meshgrid(1:size(image, 2), 1:size(image, 1));
                mask = (rowsInImage - centerY).^2 + (columnsInImage - centerX).^2 <= radius.^2;
                % Extract the RGB values within the mask
                redChannel = image(:, :, 1);
                greenChannel = image(:, :, 2);
                blueChannel = image(:, :, 3);
                redMin = 129;
                redMax = 255;
                greenMin = 108;
                greenMax = 255;
                blueMin = 111;
                blueMax = 255;
                redValues = redChannel(mask);
                rlog = redValues >= redMin & redValues <= redMax;
                redValues = redValues(rlog);
                greenValues = greenChannel(mask);
                glog = greenValues >= greenMin & greenValues <= greenMax;
                greenValues = greenValues(glog);
                blueValues = blueChannel(mask);
                blog = blueValues >= blueMin & blueValues <= blueMax;
                blueValues = blueValues(blog);
                % Calculate the average RGB values for the current zone
                averageRGB(k, 1) = mean(redValues);
                averageRGB(k, 2) = mean(greenValues);
                averageRGB(k, 3) = mean(blueValues);
                % Optional: Display the mask for visual verification
                RedChannel = image(:, :, 1);
                GreenChannel = image(:, :, 2);
                BlueChannel = image(:, :, 3);
                RedChannel(mask) = 255;
                GreenChannel(mask) = 255;
                BlueChannel(mask) = 255;
                image(:, :, 1) = RedChannel;
                image(:, :, 2) = GreenChannel;
                image(:, :, 3) = BlueChannel;
            end
            RGB = averageRGB;
        end

    end

    % App initialization and construction
    methods (Access = public)

        % Construct app
        function app = ZincConcentrationApp

            % Create UIFigure and components
            createComponents(app)

            % Register the app with App Designer
            registerApp(app, app.UIFigure)

            % Execute the startup function
            runStartupFcn(app, @startupFcn)
        end

        % Code that executes before app deletion
        function delete(app)

            % Delete UIFigure when app is deleted
            delete(app.UIFigure)
        end
    end

    methods (Access = private)

        % Create UIFigure and components
        function createComponents(app)

            % Create UIFigure
            app.UIFigure = uifigure;
            app.UIFigure.Position = [100 100 640 480];
            app.UIFigure.Name = 'Zinc Concentration App';
            app.UIFigure.Color = [0.8 0.8 0.8]; % Set grey background color

            % Create UploadBaseImagesButton
            app.UploadBaseImagesButton = uibutton(app.UIFigure, 'push');
            app.UploadBaseImagesButton.ButtonPushedFcn = createCallbackFcn(app, @uploadBaseImages, true);
            app.UploadBaseImagesButton.Position = [25 425 150 30];
            app.UploadBaseImagesButton.Text = 'Upload Base Images';

            % Create UploadReactionImagesButton
            app.UploadReactionImagesButton = uibutton(app.UIFigure, 'push');
            app.UploadReactionImagesButton.ButtonPushedFcn = createCallbackFcn(app, @uploadReactionImages, true);
            app.UploadReactionImagesButton.Position = [200 425 150 30];
            app.UploadReactionImagesButton.Text = 'Upload Reaction Images';

            % Create BaseImagesTable
            app.BaseImagesTable = uitable(app.UIFigure);
            app.BaseImagesTable.ColumnName = {'Base Images'};
            app.BaseImagesTable.RowName = {};
            app.BaseImagesTable.Position = [25 275 250 125];

            % Create ReactionImagesTable
            app.ReactionImagesTable = uitable(app.UIFigure);
            app.ReactionImagesTable.ColumnName = {'Reaction Images'};
            app.ReactionImagesTable.RowName = {};
            app.ReactionImagesTable.Position = [325 275 250 125];

            % Create CalculateZincConcentrationButton
            app.CalculateZincConcentrationButton = uibutton(app.UIFigure, 'push');
            app.CalculateZincConcentrationButton.ButtonPushedFcn = createCallbackFcn(app, @calculateZincConcentration, true);
            app.CalculateZincConcentrationButton.Position = [25 200 200 30];
            app.CalculateZincConcentrationButton.Text = 'Calculate Zinc Concentration';

            % Create ZincConcentrationTextArea
            app.ZincConcentrationTextArea = uitextarea(app.UIFigure);
            app.ZincConcentrationTextArea.Position = [25 150 100 30];
            app.ZincConcentrationTextArea.Editable = 'off';
            
            % Create IntensityTextArea
            app.IntensityTextAreaBase = uitextarea(app.UIFigure);
            app.IntensityTextAreaBase.Position = [150 150 100 30];
            app.IntensityTextAreaBase.Editable = 'off';
            
            % Create ZincConcentrationLabel
            app.ZincConcentrationLabel = uilabel(app.UIFigure);  % Add this line
            app.ZincConcentrationLabel.Text = 'Zinc Concentration';  % Set the label text
            app.ZincConcentrationLabel.Position = [25 120 150 30];  % Set the position
            
           % Create IntensityLabel
            app.ZincConcentrationLabel = uilabel(app.UIFigure);  % Add this line
            app.ZincConcentrationLabel.Text = 'Intensity';  % Set the label text
            app.ZincConcentrationLabel.Position = [150 120 150 30];  % Set the position           
          
            % Create StatusLamp
            app.StatusLamp = uilamp(app.UIFigure);
            app.StatusLamp.Position = [260 200 20 20];

           % Create ZincGauge
            app.ZincGauge = uigauge(app.UIFigure, 'linear');
            app.ZincGauge.Limits = [0 20];
            app.ZincGauge.Position = [300 200 300 30];
            app.StatusLamp.Color = [1 1 1]; % Set initial lamp color
            
             % Create StatusTextBox
            app.StatusTextBox = uitextarea(app.UIFigure);
            app.StatusTextBox.Editable = 'off';
            app.StatusTextBox.Position = [300 150 200 30];  
        end
    end
end
