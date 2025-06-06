private static void extractEventId(File file, Environment environment) {
    try (BufferedReader reader = new BufferedReader(new FileReader(file))) {
        String line;

        while ((line = reader.readLine()) != null) {
            if (line.toLowerCase().contains("event_id")) {
                Matcher matcher = Pattern.compile(":=\\s*\"([^\"]+)\"").matcher(line);
                if (matcher.find()) {
                    String eventId = matcher.group(1);
                    System.out.println("🔍 Extracted eventId: " + eventId);

                    InputStream is = null;
                    try {
                        IHttpInterface sender = HttpFactory.createInstance(
                            environment,
                            IRegulatoryConstants.REPORTING_SERVICE_EVENT_HIERARCHY_API,
                            HttpInterfaceType.GET_INTERFACE
                        );

                        is = sender.execute(eventId);

                        if (is == null) {
                            System.out.println("❌ No response from RS server for eventId: " + eventId);
                            continue; // Skip to next line
                        }

                        ZipScenarioFile zipScenarioActual = new ZipScenarioFile();
                        try {
                            zipScenarioActual.setZipBytes(ProcessXMLFile.processingZip(IOUtils.toByteArray(is)));
                        } catch (IOException e) {
                            System.out.println("❌ Failed to process ZIP for eventId: " + eventId);
                            e.printStackTrace();
                            continue;
                        }

                        String parentDir = file.getParent();
                        String originalName = file.getName();
                        String zipFileName = originalName.replace("_out.pdl", "_out.zip");

                        File targetZipFile = new File(parentDir, zipFileName);
                        try (FileOutputStream fos = new FileOutputStream(targetZipFile)) {
                            fos.write(zipScenarioActual.getZipBytes());
                            System.out.println("✅ Saved ZIP file: " + targetZipFile.getAbsolutePath());
                        } catch (IOException e) {
                            System.out.println("❌ Failed to write ZIP file for eventId: " + eventId);
                            e.printStackTrace();
                        }

                    } catch (Exception e) {
                        System.out.println("❌ Error calling RS server for eventId: " + eventId);
                        e.printStackTrace();
                        continue;
                    } finally {
                        if (is != null) {
                            try {
                                is.close();
                            } catch (IOException e) {
                                System.out.println("⚠️ Failed to close InputStream.");
                            }
                        }
                    }
                }
            }
        }
    } catch (IOException e) {
        System.out.println("❌ Error reading file: " + file.getAbsolutePath());
        e.printStackTrace();
    }
}
