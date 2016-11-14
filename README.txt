Detailed error message is not displayed on many screens - only "An error has occurred" is displayed


As part of field validation for Admin Console screens, typically on a "Save" operation, a detailed error message may be displayed on the screen if an invalid field value has been specified.
For example, this may occur if the field value is out-of-range, conflicts with something else or has invalid contents etc.
I am specifically referring to the case when the value is not validated by Javascript on the page, but rather found to be invalid by a Glassfish internal RESTful web-service used for processing the field data.

What is noticeable in Glassfish 4.1 and above, is that for screen field values that fail validation by Glassfish internal web-services, rather than on-page validation by Javascript, in many cases the detailed error message is MISSING. Only "! An error has occurred" is displayed (which is insufficient in order to understand what error actually occurred). The detailed error message, which normally is displayed under this, is not displayed. Note that this problem didn't exist in GlassfishV3 (at worst, the detailed error message was "Check server log for more information").

Simple example cases of this "missing detailed error message" issue occur with integer-based fields, for example:

\(i) "Deployment Order" for "Edit Connector Resource" screen (use value: 9999999999)
(ii) "Buffer Size" on the "Edit Transport" screen (use value: 8192999999)


You could argue that these screens should be fixed to consistently validate field data on-page using Javascript only, in these particular cases (and they probably should), however there are some things that can ONLY be validated by sending them back to Glassfish. In this case, when validation fails, a detailed error message is MEANT to be returned and displayed on the screen.

The missing detailed error message results from a combination of bugs in Glassfish's REST error-handling code, in the "org.glassfish.admin.rest.resources.TemplateRestResource" class:

1) This class was modified in Glassfish4 to throw WebApplicationExceptions instead of returning Responses. However, in the "doCreateOrUpdate" method, there already is an outer catch block that will catch these exceptions and wrap them in ANOTHER WebApplicationException (with code INTERNAL_SERVER_ERROR and blank error message), and this effectively hides the inner error message, when the outer WebApplicationException is caught and processed at a higher level.
2) The "handleError" method builds a Response with a plain-text message, which is contrary to the response header content type (json), and the resulting response cannot be parsed (conversion of json to plain text fails).

I have constructed the following 4.1.1 patch to correct these two issues (same patch code will work with Glassfish5):

{code}

-- TemplateRestResource.java	(revision 64111)
+++ TemplateRestResource.java	(working copy)
@@ -234,16 +234,18 @@
                 return new RestActionReporter();
             }
             //just update it.
             data = ResourceUtil.translateCamelCasedNamesToXMLNames(data);
             RestActionReporter ar = Util.applyChanges(data, uriInfo, getSubject());
             if (ar.getActionExitCode() != ActionReport.ExitCode.SUCCESS) {
-                throwError(Status.BAD_REQUEST, "Could not apply changes" + ar.getMessage()); // i18n
+                throwError(Status.BAD_REQUEST, "Could not apply changes: " + ar.getMessage()); // i18n
             }
 
             return ar;
+        } catch (WebApplicationException wae) {
+            throw wae;
         } catch (Exception ex) {
             throw new WebApplicationException(ex, Response.Status.INTERNAL_SERVER_ERROR);
         }
     }
 
     protected ExitCode doDelete(HashMap<String, String> data) {
@@ -634,9 +636,16 @@
         throw new WebApplicationException(handleError(error, message));
     }
 
     protected Response handleError(final Status error, final String message) throws WebApplicationException {
         //TODO better error handling.
 //                return Response.status(400).entity(ResourceUtil.getActionReportResult(ar, "Could not apply changes" + ar.getMessage(), requestHeaders, uriInfo)).build();
-        return Response.status(error).entity(message).build();
+        //return Response.status(error).entity(message).build();
+        // BUG-FIX
+        // The original line of code (commented-out below) builds a Response with a plain-text message, which is contrary to the response header content type (json).
+        // The resulting response cannot be parsed (conversion of json to plain text fails).
+        // This response parsing error never happened in the original GF code because the thrown WebApplicationException was
+        // erroneously wrapped with another WebApplicationException which had an INTERNAL_SERVER_ERROR status code and a blamk message.
+        // return Response.status(error).entity(message).build();
+        return Response.status(error).entity(ResourceUtil.getActionReportResult(ActionReport.ExitCode.FAILURE, message, requestHeaders, uriInfo)).build();
     }
 }

{code}

With this patch applied, an additional detailed message is displayed, like the following, for example case \(i) mentioned above:

{panel:bgColor=#FFFFCE}
    Could not apply changes: Could not change the attributes: javax.validation.ConstraintViolationException: Constraints for this ConnectorResource configuration have been violated: on property [ org.jvnet.hk2.config.WriteableView$4$2@726e42e9 ] violation reason [ is not of data type:java.lang.Integer ]
{panel}

There still is a minor problem though. In the above error message you will notice that the property name is "org.jvnet.hk2.config.WriteableView$4$2@726e42e9", rather than the actual property name.
This appears to be because of a problem in one of the HK2 configuration classes (org.jvnet.hk2.config.WriteableView) in “hk2-config.jar”. The code in question is shown below, with the problem line highlighted using ★★.

{code}
private void handleValidationException(Set constraintViolations) throws ConstraintViolationException {

        if (constraintViolations != null && !constraintViolations.isEmpty()) {
            Iterator<ConstraintViolation<ConfigBeanProxy>> it = constraintViolations.iterator();

            StringBuilder sb = new StringBuilder();
            sb.append(MessageFormat.format(i18n.getString("bean.validation.failure"), this.<ConfigBeanProxy>getProxyType().getSimpleName()));
            String violationMsg = i18n.getString("bean.validation.constraintViolation");
            while (it.hasNext()) {
                ConstraintViolation cv = it.next();
                sb.append(" ");
                sb.append(MessageFormat.format(violationMsg, cv.getMessage(), cv.getPropertyPath()));  //★★
                if (it.hasNext()) {
                    sb.append(i18n.getString("bean.validation.separator"));
                }
            }
            bean.getLock().unlock();
            throw new ConstraintViolationException(sb.toString(), constraintViolations);
        }
}

{code}

★★The call to “cv.getPropertyPath()” above is returning a "javax.validation.Path" instance, so the code is relying on the toString() method of the implementation class to supply the property name string, but no such toString() method has been defined. Therefore the call to “cv.getPropertyPath()” above should instead be  “cv.getPropertyPath().iterator().next().getName()”.
Or, alternatively, the code above could be left unchanged, and instead a “toString()” method could be added to the Path implementation, also defined in the WriteableView class, as shown below:

{code}

                    return new javax.validation.Path() {
                        @Override
                        public Iterator<Node> iterator() {
                            return nodes.iterator();
                        }
                        @Override
                        public String toString() {  //★★
                            return nodes.iterator().next().getName();
                        }
                    };

{code}

With the above fix applied, the displayed error message then correctly includes the actual property name, as shown below:

{panel:bgColor=#FFFFCE}
    Could not apply changes: Could not change the attributes: javax.validation.ConstraintViolationException: Constraints for this ConnectorResource configuration have been violated: on property [ deployment-order ] violation reason [ is not of data type:java.lang.Integer ]
{panel}

